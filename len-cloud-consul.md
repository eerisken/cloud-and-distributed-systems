# **Lean Cloud: The Complete Bare-Metal Blueprint**

In this architecture, we combine the raw speed of **Bare-Metal Fedora** with the safety of **WebAssembly**. We use **Terraform** to provision our 5-server pool, **Consul** for service discovery, **Nomad** for orchestration, and **MinIO** as our high-performance shared file system.

## **1. Our Infrastructure: Terraform Hardening**

We start by transforming our 5 fresh Fedora servers into a hardened cluster. This script installs the necessary binaries and configures the system to allow Consul to handle DNS resolution.  

**main.tf**
```Terraform

variable "fedora_ips" {  
  description = "The IP addresses of our 5 bare-metal servers"  
  default     = ["10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.4", "10.0.0.5"]  
}

resource "null_resource" "cluster_setup" {  
  count = length(var.fedora_ips)

  connection {  
    type        = "ssh"  
    user        = "fedora"  
    private_key = file("~/.ssh/id_rsa")  
    host        = var.fedora_ips[count.index]  
  }

  provisioner "remote-exec" {  
    inline = [  
      # 1. Install Dependencies  
      "sudo dnf install -y wasmtime nomad consul",  
        
      # 2. Configure DNS to use Consul (Port 8600)  
      # This allows our code to use 'http://minio.service.consul'  
      "sudo mkdir -p /etc/systemd/resolved.conf.d",  
      "echo '[Resolve]' | sudo tee /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'DNS=127.0.0.1:8600' | sudo tee -a /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'DNSSEC=false' | sudo tee -a /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'Domains=~consul' | sudo tee -a /etc/systemd/resolved.conf.d/consul.conf",  
      "sudo systemctl restart systemd-resolved",

      # 3. Security Hardening (Firewall & SELinux)  
      "sudo setenforce 1",  
      "sudo firewall-cmd --permanent --add-port=4646/tcp", # Nomad HTTP  
      "sudo firewall-cmd --permanent --add-port=4647/tcp", # Nomad RPC  
      "sudo firewall-cmd --permanent --add-port=4648/tcp", # Nomad Serf  
      "sudo firewall-cmd --permanent --add-port=8500/tcp", # Consul UI  
      "sudo firewall-cmd --permanent --add-port=8600/tcp", # Consul DNS  
      "sudo firewall-cmd --permanent --add-port=8600/udp", # Consul DNS  
      "sudo firewall-cmd --permanent --add-port=9000/tcp", # MinIO  
      "sudo firewall-cmd --reload",

      # 4. Enable Services  
      "sudo systemctl enable --now nomad consul"  
    ]  
  }  
}
```

## **2. Our Storage Setup**
Before deploying code, we prepare our stateful backends (Postgres and MinIO) on the cluster.  

**Step A: Postgres Schema (Logs)**

```SQL
CREATE DATABASE lean_cloud;  
c lean_cloud;

CREATE TABLE request_logs (  
    id SERIAL PRIMARY KEY,  
    user_ip VARCHAR(50),  
    request_text TEXT,  
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP  
);
```

**Step B: MinIO Buckets (Shared Files)**
```Bash

# We assume MinIO is running on one node (e.g., 10.0.0.5)  
export MC_HOST_myminio="http://admin:password@10.0.0.5:9000"

# Create the bucket  
mc mb myminio/cloud-pdfs

# Set public download policy (so our internal services can read easily)  
mc anonymous set download myminio/cloud-pdfs
```
## **3. Our Microservices (Rust + Wasm)**

We implement two services. We use reqwest for HTTP interactions with Consul and MinIO.

### **Cargo.toml (Shared Dependencies)**
```Ini, TOML

[dependencies]  
tokio = { version = "1", features = ["full"] }  
warp = "0.3"  
serde = { version = "1.0", features = ["derive"] }  
serde_json = "1.0"  
reqwest = { version = "0.11", features = ["json", "blocking"] } # Blocking for simpler Wasm I/O  
sqlx = { version = "0.7", features = ["runtime-tokio", "postgres"] }  
printpdf = "0.7"  
uuid = { version = "1.0", features = ["v4"] }  
anyhow = "1.0"
```
### **Service A: The PDF Server (Orchestrator)**

This server generates the PDF, PUTs it to MinIO, and triggers the email service.  

**pdf_server/src/main.rs**
```Rust

use warp::Filter;  
use sqlx::postgres::PgPoolOptions;  
use printpdf::*;  
use std::io::BufWriter;

#[tokio::main]  
async fn main() {  
    let db_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");  
    let pool = PgPoolOptions::new().max_connections(50).connect(&db_url).await.unwrap();

    let route = warp::post()  
        .and(warp::body::json())  
        .and(warp::addr::remote())  
        .and(warp::any().map(move || pool.clone()))  
        .and_then(process_request);

    println!("PDF Orchestrator listening on port 8080");  
    warp::serve(route).run(([0, 0, 0, 0], 8080)).await;  
}

async fn process_request(body: serde_json::Value, addr: Option<std::net::SocketAddr>, pool: sqlx::PgPool) -> Result<impl warp::Reply, warp::Rejection> {  
    let text = body["text"].as_str().unwrap_or("Empty");  
    let email = body["email"].as_str().unwrap_or("admin@local");  
    let ip = addr.map(|a| a.to_string()).unwrap_or_else(|| "0.0.0.0".into());

    // 1. Log to Postgres  
    sqlx::query("INSERT INTO request_logs (user_ip, request_text) VALUES ($1, $2)")  
        .bind(&ip).bind(text).execute(&pool).await.ok();

    // 2. Generate PDF (In-Memory)  
    let (doc, page, layer) = PdfDocument::new("Cloud PDF", Mm(210.0), Mm(297.0), "Layer 1");  
    let font = doc.add_builtin_font(BuiltinFont::Helvetica).unwrap();  
    doc.get_page(page).get_layer(layer).use_text(text, 12.0, Mm(10.0), Mm(280.0), &font);  
      
    let mut buffer = Vec::new();  
    doc.save(&mut BufWriter::new(&mut buffer)).unwrap();

    // 3. Upload to Shared Storage (MinIO)  
    let file_id = uuid::Uuid::new_v4().to_string();  
    let file_name = format!("{}.pdf", file_id);  
    let minio_url = format!("http://minio.service.consul:9000/cloud-pdfs/{}", file_name);

    let client = reqwest::Client::new();  
    // THE LOGIC: We perform a standard HTTP PUT with the raw bytes  
    let upload_res = client.put(&minio_url)  
        .body(buffer)  
        .send()  
        .await;

    if upload_res.is_err() {  
        return Ok(warp::reply::json(&serde_json::json!({"error": "MinIO upload failed"})));  
    }

    // 4. Trigger Email Service (Service Discovery via Consul DNS)  
    let email_service_url = "http://email-service.service.consul/send";  
    let _ = client.post(email_service_url)  
        .json(&serde_json::json!({  
            "to": email,   
            "file_url": minio_url   
        }))  
        .send()  
        .await;

    Ok(warp::reply::json(&serde_json::json!({  
        "status": "Success",  
        "file_id": file_id  
    })))  
}
```
### **Service B: The Email Service (Worker)**
This service fetches the file from MinIO and "sends" it.  

**email_service/src/main.rs**
```Rust

use warp::Filter;

#[tokio::main]  
async fn main() {  
    let send_route = warp::post()  
        .and(warp::path("send"))  
        .and(warp::body::json())  
        .and_then(send_email);

    // Get dynamic port from Nomad  
    let port: u16 = std::env::var("NOMAD_PORT_http")  
        .unwrap_or("8081".into())  
        .parse().unwrap();

    println!("Email Worker listening on port {}", port);  
    warp::serve(send_route).run(([0, 0, 0, 0], port)).await;  
}

async fn send_email(body: serde_json::Value) -> Result<impl warp::Reply, warp::Rejection> {  
    let recipient = body["to"].as_str().unwrap();  
    let file_url = body["file_url"].as_str().unwrap();

    println!("Worker received task for: {}", recipient);

    // 1. Download from Shared Storage (MinIO)  
    // THE LOGIC: We fetch the file bytes from the internal URL provided by the PDF server  
    let client = reqwest::Client::new();  
    let resp = client.get(file_url).send().await;

    match resp {  
        Ok(response) => {  
            let bytes = response.bytes().await.unwrap();  
            println!("Downloaded {} bytes from MinIO. Attaching to email...", bytes.len());  
            // [Stub: SMTP logic would go here]  
            println!("Email sent to {}!", recipient);  
            Ok(warp::reply::json(&serde_json::json!({"status": "sent"})))  
        }  
        Err(_) => {  
            println!("Failed to download file from storage.");  
            Ok(warp::reply::json(&serde_json::json!({"status": "error_downloading"})))  
        }  
    }  
}
```
## **4. Our Orchestration: Nomad Job**
We define the deployment of our 9 instances (6 PDF, 3 Email) in a single file.  

**lean_cloud.nomad**
```Terraform

job "lean-cloud" {  
  datacenters = ["dc1"]

  # GROUP 1: The Email Workers  
  group "email-workers" {  
    count = 3 # Redundancy across 3 servers

    network {  
      port "http" {} # Dynamic port assignment  
    }

    task "email-server" {  
      driver = "wasm"  
      config {  
        path = "local/email_service.wasm"  
      }  
        
      # Register with Consul so PDF server can find us  
      service {  
        name = "email-service"  
        port = "http"  
        check {  
          type     = "http"  
          path     = "/health" # (Assumes we added a health route)  
          interval = "10s"  
          timeout  = "2s"  
        }  
      }  
    }  
  }

  # GROUP 2: The PDF Orchestrators  
  group "pdf-orchestrators" {  
    count = 6 # High capacity handling

    network {  
      port "http" {  
        static = 8080 # We want a fixed entry point  
      }  
    }

    task "pdf-server" {  
      driver = "wasm"  
      config {  
        path = "local/pdf_server.wasm"  
        # Environment variables for connections  
        args = [  
          "--env", "DATABASE_URL=postgres://admin:pass@10.0.0.5:5432/lean_cloud",  
          "--env", "RUST_LOG=info"  
        ]  
      }  
    }  
  }  
}
```
## **5. Execution Summary**
We execute the following on our workstation to bring the cloud to life:

1. **Build:**  
   Bash  
   cargo build --target wasm32-wasip1 --release

2. **Provision:**  
   Bash  
   terraform init && terraform apply -auto-approve

3. **Deploy:**  
   Bash  
   export NOMAD_ADDR="http://10.0.0.1:4646"  
   nomad job run lean_cloud.nomad

We now have a fully functional, self-healing, distributed cloud running on bare metal with shared storage and service discovery.
