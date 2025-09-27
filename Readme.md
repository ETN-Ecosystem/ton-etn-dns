

You can run the necessary TON Proxy component in its own LXC container and configure it to work alongside or in front of your existing Caddy setup to serve multiple apps via different TON domains.

Here's the breakdown of how to adapt your self-hosted architecture for the TON Network:

-----

## 1\. The Core Concept: A Two-Part Reverse Proxy

Your existing setup already uses a reverse proxy chain, and we'll essentially be adding a new layer on the TON side.

  * **TON Layer (RLDP → HTTP):** This will be a new LXC container running the **TON Proxy** application (`rldp-http-proxy` or an alternative like `tonutils-reverse-proxy`). Its job is to listen for TON Network requests (RLDP/ADNL) and convert them into standard HTTP requests.
  * **Existing Caddy Layer (HTTP → Internal Apps):** This is your current Caddy LXC. The TON Proxy will forward the new HTTP requests to this Caddy instance. Caddy will then route them to the correct internal LXC based on the domain name, just like it does for the regular internet.

-----

## 2\. Setup the Dedicated TON Proxy LXC

You should create a new LXC container to run the TON-specific reverse proxy component. Given your existing setup, this keeps the TON network-facing software isolated.

### A. TON Proxy Configuration

1.  **Install the Proxy:** Download and install the TON reverse proxy software (e.g., `rldp-http-proxy` or `tonutils-reverse-proxy`) inside the new LXC container.

2.  **Generate ADNL Address:** Generate a persistent **ADNL address** for this LXC. This single address is what all of your TON domains (`app1.ton`, `app2.ton`, etc.) will point to via TON DNS.

3.  **Run in Reverse Mode:** Configure the proxy to run in **reverse mode**. Instead of forwarding to a single web server on `127.0.0.1:80`, you'll set it up to forward all incoming TON requests to your existing Caddy LXC.

      * For the standard `rldp-http-proxy`, you'd use the `-R` flag to point to your Caddy container's **internal IP** and **port** (likely port 80/443, or whatever Caddy is listening on internally for the LAN).

    **Example Command (Conceptual):**

    ```bash
    # Replace <CADDY_LXC_IP> with the internal network IP of your Caddy LXC
    rldp-http-proxy -a <your-public-ip>:3333 -R '*'@<CADDY_LXC_IP>:80 -C global.config.json -A <your-adnl-address> -d
    ```

### B. DNS Setup

1.  **Register Domains:** Acquire the `.ton` domains you need (e.g., `etn-app1.ton`, `etn-app2.ton`).
2.  **Create Site Records:** For *each* `.ton` domain, you must register a **site record** via TON DNS that points to the **single ADNL address** of your new TON Proxy LXC.

-----

## 3\. Configure Your Existing Caddy LXC

This is the most critical step for handling the multiple TON domains. Your Caddy configuration needs to recognize and respond to the `.ton` domains when they arrive as HTTP requests from the TON Proxy LXC.

Since the TON Proxy converts the RLDP/ADNL request into a standard HTTP request, the **Host header** will be preserved. This means the request that hits your Caddy instance will look like this:

`GET / HTTP/1.1`
`Host: etn-app1.ton`
`...`

### Caddy Configuration

In your `Caddyfile` on the Caddy LXC, you simply add server blocks for the new TON domains and proxy them to their corresponding internal LXC containers (apps).

```caddy
# This is where the request from the regular internet comes in
# your.main-domain.com {
#   reverse_proxy 10.0.0.X:8080 
#   ...
# }

# NEW: Configuration for TON Site 1
# This will be routed from the TON Proxy LXC based on the Host header
etn-app1.ton {
    # Point to the internal IP and port of the LXC running App 1
    reverse_proxy 10.0.0.11:80
}

# NEW: Configuration for TON Site 2
# (The TON Proxy LXC sends all TON traffic to Caddy, Caddy decides where to route it)
etn-app2.ton {
    # Point to the internal IP and port of the LXC running App 2
    reverse_proxy 10.0.0.12:443  # Example: if App 2 uses HTTPS internally
}
```

By using the TON Proxy to forward all requests to your existing Caddy instance, you leverage Caddy's built-in **Virtual Host** capability to route multiple distinct `.ton` domains to all your different self-hosted applications.

This setup keeps your core **self-hosting** principle intact: everything is run and managed on your own Proxmox environment, with the TON component isolated in its own container.3.  **Caddy Configuration (The Crux):** This is where you configure the specific routing for each customer's external URL.

The only difference is that your Caddy config will use an **external, public Web2 domain** as the backend target instead of a local LXC IP address.

-----

## Step-by-Step Service Implementation

### 1\. The Customer Interface (Your Service)

You would need a system to manage your customer's requests.

| Requirement | Implementation/Action |
| :--- | :--- |
| **Input** | A customer provides their **Web2 URL** (e.g., `https://www.customer-shop.com`). |
| **Domain Link** | The customer pays you for a new **`.ton` domain** (e.g., `customer-shop.ton`). |
| **DNS Update** | You register or update the TON DNS record for `customer-shop.ton` to point to your **public ADNL address**. This is the one you generated for your TON Proxy LXC. |

### 2\. Caddy Reverse Proxy Configuration

You will set up your Caddyfile to include a new block for every customer. Caddy is powerful because it can fetch content from *any* HTTP/S endpoint.

The configuration for a new customer would look like this in your Caddyfile:

```caddy
# This is the domain the user is visiting on the TON Network
customer-shop.ton {
    # The reverse_proxy directive tells Caddy where to get the content from.
    # In this case, it's the customer's live Web2 URL on the public internet.
    reverse_proxy https://www.customer-shop.com {
        # Optional: Add headers to hide the fact that you are proxying
        header_up Host {http.request.host} # Pass the TON domain as the Host header (usually)
        # OR:
        # header_up Host www.customer-shop.com # Pass the external domain as the Host header (if needed by their server)
    }
}
```

By adding a new block for each `.ton` domain and pointing its `reverse_proxy` directive to the customer's Web2 URL, you can scale this service indefinitely—all behind your **single ADNL address and private key**.

### 3\. Key Considerations for a Service

  * **HTTPS/SSL on Web2:** Note that the reverse proxy should connect to the Web2 site using **HTTPS** (`https://...`). Your Caddy/TON Proxy does not need to handle the Web2 site's certificate; it just acts as a client to the external secure server. The connection **within TON** is secured by ADNL's encryption, so the user sees a secure experience.
  * **Performance:** All traffic from all customer `.ton` sites will flow through your **single** TON Proxy LXC. You will need a robust host and reliable network bandwidth to handle the aggregate load, especially as your service grows.
  * **Web2 Content Compatibility:** Most modern websites work fine, but some Web2 sites use JavaScript that checks the domain name to ensure it matches the certificate. You may need to inject or modify headers to ensure the customer's Web2 server is happy serving content when the `Host` header says `customer-shop.ton`.
