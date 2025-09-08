# Ingesting Docker Container Logs into Splunk with Portainer

> This walkthrough assumes you already have Splunk up and running and will demonstrate how to configure Docker containers to ingest logs into Splunk Enterprise through HTTP Event Collector (HEC) over port 8088
> 


![image.png](../screenshots/image.png)


## **Prerequisites**

- Splunk Enterprise running in your lab.
- Admin access to Splunk Web.
- Portainer managing your Docker environment.
- Basic Docker knowledge.


## **Step 1: Enable Splunk HTTP Event Collector (HEC)**

1. Log into **Splunk Web**.
2. Navigate: **Settings → Data Inputs → HTTP Event Collector**.
3. Click **Global Settings** → Enable **All Tokens**.
4. Note the **HTTP Port Number** (default 8088).
5. Create a new token:
    - Name: docker-logs
    - Allowed Index: dvwa (or create a dedicated index)
    - **Disable Indexer Acknowledgment** (important — Docker logging driver does not set channels).
6. Copy the generated **HEC Token**.
    
    ![Screenshot 2025-08-31 at 8.21.48 PM.png](../screenshots/Screenshot_2025-08-31_at_8.21.48_PM.png)
    


## **Step 2: Verify HEC**

Run a quick test from the Docker host:

```
curl -sk https://<splunk-ip>:8088/services/collector/event \
  -H "Authorization: Splunk <HEC_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"event":"hec test event","sourcetype":"hec:test","index":"dvwa"}'
```

Expected response:

```
{"text":"Success","code":0}
```

Check in Splunk Search:

```
index=dvwa sourcetype=hec:test
```

![image1.png](../screenshots/image1.png)


## **Step 3: Configure Container Logging in Portainer**

1. In Portainer, open **Containers** → Select your container → **Duplicate/Edit**.
2. Go to **Advanced container settings → Logging**.
3. Configure:
    - **Driver:** splunk
    - **Options:**
        - splunk-url → https://<splunk-ip>:8088
        - splunk-token → <HEC_TOKEN>
        - splunk-insecureskipverify → true (lab only)
        - splunk-index → dvwa
        - splunk-sourcetype → docker:json
        - splunk-format → json
        - tag → {{.Name}}
    
    ![image2.png](../screenshots/image2.png)
    
4. Redeploy the container.


## **Step 4: Confirm Logs in Splunk**

Generate activity in DVWA (login, click around).

Then search in Splunk:

```
index=dvwa sourcetype=docker:json
```

![image3.png](../screenshots/image3.png)


## **Step 5: Scale with Portainer Stacks**

Instead of editing each container, apply logging globally in a Stack:

```
version: "3.8"

services:
  dvwa:
    image: vulnerables/web-dvwa
    container_name: dvwa
    ports:
      - "80:80"
    logging:
      driver: splunk
      options:
        splunk-url: https://splunk.lab.local:8088
        splunk-token: ${SPLUNK_HEC_TOKEN}
        splunk-insecureskipverify: "true"
        splunk-index: dvwa
        splunk-sourcetype: docker:json
        splunk-format: json
        tag: "{{.Name}}"

  nginx:
    image: nginx:alpine
    container_name: nginx
    logging:
      driver: splunk
      options:
        splunk-url: https://splunk.lab.local:8088
        splunk-token: ${SPLUNK_HEC_TOKEN}
        splunk-insecureskipverify: "true"
        splunk-index: infra
        splunk-sourcetype: docker:json
        splunk-format: json
        tag: "{{.Name}}"
```

Deploy via **Portainer → Stacks → Add Stack**.


## **Troubleshooting**

- **Code 10 “Data channel is missing”** → Disable Indexer Acknowledgment for your HEC token.
- **No logs** → Verify container logs via docker logs <container> (some apps log to files instead of stdout/stderr).
- **Connection errors** → Confirm firewall (TCP 8088) and Splunk HEC enabled.
- **Wrong index** → Make sure the token is allowed to write to the chosen index.


## **Outcome**

- DVWA and other Docker containers now stream logs directly into Splunk.
- Logs are searchable by container name, index, and sourcetype.
- Portainer makes scaling log collection across multiple containers straightforward.
