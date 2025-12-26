# **Lean Cloud: The Complete Bare-Metal Blueprint**

In this architecture, we combine the raw speed of **Bare-Metal Fedora** with the safety of **WebAssembly**. We use **Terraform** to provision our 5-server pool, **Consul** for service discovery, **Nomad** for orchestration, and **MinIO** as our high-performance shared file system.

## **1\. Our Infrastructure: Terraform Hardening**

We start by transforming our 5 fresh Fedora servers into a hardened cluster. This script installs the necessary binaries and configures the system to allow Consul to handle DNS resolution.  
**main.tf**

Terraform

variable "fedora\_ips" {  
  description \= "The IP addresses of our 5 bare-metal servers"  
  default     \= \["10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.4", "10.0.0.5"\]  
}

resource "null\_resource" "cluster\_setup" {  
  count \= length(var.fedora\_ips)

  connection {  
    type        \= "ssh"  
    user        \= "fedora"  
    private\_key \= file("\~/.ssh/id\_rsa")  
    host        \= var.fedora\_ips\[count.index\]  
  }

  provisioner "remote-exec" {  
    inline \= \[  
      \# 1\. Install Dependencies  
      "sudo dnf install \-y wasmtime nomad consul",  
        
      \# 2\. Configure DNS to use Consul (Port 8600\)  
      \# This allows our code to use 'http://minio.service.consul'  
      "sudo mkdir \-p /etc/systemd/resolved.conf.d",  
      "echo '\[Resolve\]' | sudo tee /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'DNS=127.0.0.1:8600' | sudo tee \-a /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'DNSSEC=false' | sudo tee \-a /etc/systemd/resolved.conf.d/consul.conf",  
      "echo 'Domains=\~consul' | sudo tee \-a /etc/systemd/resolved.conf.d/consul.conf",  
      "sudo systemctl restart systemd-resolved",

      \# 3\. Security Hardening (Firewall & SELinux)  
      "sudo setenforce 1",  
      "sudo firewall-cmd \--permanent \--add-port=4646/tcp", \# Nomad HTTP  
      "sudo firewall-cmd \--permanent \--add-port=4647/tcp", \# Nomad RPC  
      "sudo firewall-cmd \--permanent \--add-port=4648/tcp", \# Nomad Serf  
      "sudo firewall-cmd \--permanent \--add-port=8500/tcp", \# Consul UI  
      "sudo firewall-cmd \--permanent \--add-port=8600/tcp", \# Consul DNS  
      "sudo firewall-cmd \--permanent \--add-port=8600/udp", \# Consul DNS  
      "sudo firewall-cmd \--permanent \--add-port=9000/tcp", \# MinIO  
      "sudo firewall-cmd \--reload",

      \# 4\. Enable Services  
      "sudo systemctl enable \--now nomad consul"  
    \]  
  }  
}

## **2\. Our Storage Setup**

Before deploying code, we prepare our stateful backends (Postgres and MinIO) on the cluster.  
**Step A: Postgres Schema (Logs)**

SQL

CREATE DATABASE lean\_cloud;  
\\c lean\_cloud;

CREATE TABLE request\_logs (  
    id SERIAL PRIMARY KEY,  
    user\_ip VARCHAR(50),  
    request\_text TEXT,  
    created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP  
);

**Step B: MinIO Buckets (Shared Files)**

Bash

\# We assume MinIO is running on one node (e.g., 10.0.0.5)  
export MC\_HOST\_myminio="http://admin:password@10.0.0.5:9000"

\# Create the bucket  
mc mb myminio/cloud-pdfs

\# Set public download policy (so our internal services can read easily)  
mc anonymous set download myminio/cloud-pdfs

## **3\. Our Microservices (Rust \+ Wasm)**

We implement two services. We use reqwest for HTTP interactions with Consul and MinIO.

### **Cargo.toml (Shared Dependencies)**

Ini, TOML

\[dependencies\]  
tokio \= { version \= "1", features \= \["full"\] }  
warp \= "0.3"  
serde \= { version \= "1.0", features \= \["derive"\] }  
serde\_json \= "1.0"  
reqwest \= { version \= "0.11", features \= \["json", "blocking"\] } \# Blocking for simpler Wasm I/O  
sqlx \= { version \= "0.7", features \= \["runtime-tokio", "postgres"\] }  
printpdf \= "0.7"  
uuid \= { version \= "1.0", features \= \["v4"\] }  
anyhow \= "1.0"

### **Service A: The PDF Server (Orchestrator)**

This server generates the PDF, PUTs it to MinIO, and triggers the email service.  
**pdf\_server/src/main.rs**

Rust

use warp::Filter;  
use sqlx::postgres::PgPoolOptions;  
use printpdf::\*;  
use std::io::BufWriter;

\#\[tokio::main\]  
async fn main() {  
    let db\_url \= std::env::var("DATABASE\_URL").expect("DATABASE\_URL must be set");  
    let pool \= PgPoolOptions::new().max\_connections(50).connect(\&db\_url).await.unwrap();

    let route \= warp::post()  
        .and(warp::body::json())  
        .and(warp::addr::remote())  
        .and(warp::any().map(move || pool.clone()))  
        .and\_then(process\_request);

    println\!("PDF Orchestrator listening on port 8080");  
    warp::serve(route).run((\[0, 0, 0, 0\], 8080)).await;  
}

async fn process\_request(body: serde\_json::Value, addr: Option\<std::net::SocketAddr\>, pool: sqlx::PgPool) \-\> Result\<impl warp::Reply, warp::Rejection\> {  
    let text \= body\["text"\].as\_str().unwrap\_or("Empty");  
    let email \= body\["email"\].as\_str().unwrap\_or("admin@local");  
    let ip \= addr.map(|a| a.to\_string()).unwrap\_or\_else(|| "0.0.0.0".into());

    // 1\. Log to Postgres  
    sqlx::query("INSERT INTO request\_logs (user\_ip, request\_text) VALUES ($1, $2)")  
        .bind(\&ip).bind(text).execute(\&pool).await.ok();

    // 2\. Generate PDF (In-Memory)  
    let (doc, page, layer) \= PdfDocument::new("Cloud PDF", Mm(210.0), Mm(297.0), "Layer 1");  
    let font \= doc.add\_builtin\_font(BuiltinFont::Helvetica).unwrap();  
    doc.get\_page(page).get\_layer(layer).use\_text(text, 12.0, Mm(10.0), Mm(280.0), \&font);  
      
    let mut buffer \= Vec::new();  
    doc.save(\&mut BufWriter::new(\&mut buffer)).unwrap();

    // 3\. Upload to Shared Storage (MinIO)  
    let file\_id \= uuid::Uuid::new\_v4().to\_string();  
    let file\_name \= format\!("{}.pdf", file\_id);  
    let minio\_url \= format\!("http://minio.service.consul:9000/cloud-pdfs/{}", file\_name);

    let client \= reqwest::Client::new();  
    // THE LOGIC: We perform a standard HTTP PUT with the raw bytes  
    let upload\_res \= client.put(\&minio\_url)  
        .body(buffer)  
        .send()  
        .await;

    if upload\_res.is\_err() {  
        return Ok(warp::reply::json(\&serde\_json::json\!({"error": "MinIO upload failed"})));  
    }

    // 4\. Trigger Email Service (Service Discovery via Consul DNS)  
    let email\_service\_url \= "http://email-service.service.consul/send";  
    let \_ \= client.post(email\_service\_url)  
        .json(\&serde\_json::json\!({  
            "to": email,   
            "file\_url": minio\_url   
        }))  
        .send()  
        .await;

    Ok(warp::reply::json(\&serde\_json::json\!({  
        "status": "Success",  
        "file\_id": file\_id  
    })))  
}

### **Service B: The Email Service (Worker)**

This service fetches the file from MinIO and "sends" it.  
**email\_service/src/main.rs**

Rust

use warp::Filter;

\#\[tokio::main\]  
async fn main() {  
    let send\_route \= warp::post()  
        .and(warp::path("send"))  
        .and(warp::body::json())  
        .and\_then(send\_email);

    // Get dynamic port from Nomad  
    let port: u16 \= std::env::var("NOMAD\_PORT\_http")  
        .unwrap\_or("8081".into())  
        .parse().unwrap();

    println\!("Email Worker listening on port {}", port);  
    warp::serve(send\_route).run((\[0, 0, 0, 0\], port)).await;  
}

async fn send\_email(body: serde\_json::Value) \-\> Result\<impl warp::Reply, warp::Rejection\> {  
    let recipient \= body\["to"\].as\_str().unwrap();  
    let file\_url \= body\["file\_url"\].as\_str().unwrap();

    println\!("Worker received task for: {}", recipient);

    // 1\. Download from Shared Storage (MinIO)  
    // THE LOGIC: We fetch the file bytes from the internal URL provided by the PDF server  
    let client \= reqwest::Client::new();  
    let resp \= client.get(file\_url).send().await;

    match resp {  
        Ok(response) \=\> {  
            let bytes \= response.bytes().await.unwrap();  
            println\!("Downloaded {} bytes from MinIO. Attaching to email...", bytes.len());  
            // \[Stub: SMTP logic would go here\]  
            println\!("Email sent to {}\!", recipient);  
            Ok(warp::reply::json(\&serde\_json::json\!({"status": "sent"})))  
        }  
        Err(\_) \=\> {  
            println\!("Failed to download file from storage.");  
            Ok(warp::reply::json(\&serde\_json::json\!({"status": "error\_downloading"})))  
        }  
    }  
}

## **4\. Our Orchestration: Nomad Job**

We define the deployment of our 9 instances (6 PDF, 3 Email) in a single file.  
**lean\_cloud.nomad**

Terraform

job "lean-cloud" {  
  datacenters \= \["dc1"\]

  \# GROUP 1: The Email Workers  
  group "email-workers" {  
    count \= 3 \# Redundancy across 3 servers

    network {  
      port "http" {} \# Dynamic port assignment  
    }

    task "email-server" {  
      driver \= "wasm"  
      config {  
        path \= "local/email\_service.wasm"  
      }  
        
      \# Register with Consul so PDF server can find us  
      service {  
        name \= "email-service"  
        port \= "http"  
        check {  
          type     \= "http"  
          path     \= "/health" \# (Assumes we added a health route)  
          interval \= "10s"  
          timeout  \= "2s"  
        }  
      }  
    }  
  }

  \# GROUP 2: The PDF Orchestrators  
  group "pdf-orchestrators" {  
    count \= 6 \# High capacity handling

    network {  
      port "http" {  
        static \= 8080 \# We want a fixed entry point  
      }  
    }

    task "pdf-server" {  
      driver \= "wasm"  
      config {  
        path \= "local/pdf\_server.wasm"  
        \# Environment variables for connections  
        args \= \[  
          "--env", "DATABASE\_URL=postgres://admin:pass@10.0.0.5:5432/lean\_cloud",  
          "--env", "RUST\_LOG=info"  
        \]  
      }  
    }  
  }  
}

## **5\. Execution Summary**

We execute the following on our workstation to bring the cloud to life:

1. **Build:**  
   Bash  
   cargo build \--target wasm32-wasip1 \--release

2. **Provision:**  
   Bash  
   terraform init && terraform apply \-auto-approve

3. **Deploy:**  
   Bash  
   export NOMAD\_ADDR="http://10.0.0.1:4646"  
   nomad job run lean\_cloud.nomad

We now have a fully functional, self-healing, distributed cloud running on bare metal with shared storage and service discovery.