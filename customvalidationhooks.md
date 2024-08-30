To implement custom validation webhooks in OpenShift, you'll need to create a webhook that will intercept API requests to perform specific checks, like validating images for particular criteria. Below is a sample code that shows how to set up a custom validation webhook in Python using Flask. This webhook checks if an image in a pod spec has certain labels or if the image comes from a specific internal registry.

### Step-by-Step Guide to Implement Custom Validation Webhooks in OpenShift

1. **Create the Webhook Server**: A Python Flask application that listens for AdmissionReview requests from the Kubernetes API server.

2. **Deploy the Webhook**: Package the Python Flask app into a container image, then deploy it in OpenShift.

3. **Configure OpenShift to Use the Webhook**: Define a `ValidatingWebhookConfiguration` that points to your webhook service.

### Sample Code for Custom Validation Webhook

#### Step 1: Create the Webhook Server

Here’s a Python example using Flask to create a basic validation webhook:

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    request_info = request.json

    # Extract the image from the AdmissionReview request
    image = request_info['request']['object']['spec']['containers'][0]['image']
    
    # Perform validation - Check if the image is from an internal registry
    allowed_registry = "internal.registry.io"
    is_valid = image.startswith(allowed_registry)

    # Create AdmissionReview response
    admission_response = {
        "response": {
            "uid": request_info['request']['uid'],
            "allowed": is_valid,
            "status": {
                "code": 200 if is_valid else 403,
                "message": "Image is allowed." if is_valid else "Image must be from internal registry."
            }
        }
    }

    return jsonify(admission_response)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=443, ssl_context=('cert.pem', 'key.pem'))
```

#### Step 2: Deploy the Webhook in OpenShift

1. **Containerize the Webhook Server**:
   
   - Create a `Dockerfile`:
     ```dockerfile
     FROM python:3.8-slim

     WORKDIR /app

     COPY requirements.txt .
     RUN pip install -r requirements.txt

     COPY . .

     CMD ["python", "webhook.py"]
     ```

   - Build and push the image to your registry:
     ```bash
     docker build -t your-registry/webhook:v1 .
     docker push your-registry/webhook:v1
     ```

2. **Deploy the Webhook in OpenShift**:
   
   Create a deployment and service for your webhook:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: image-validation-webhook
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: image-validation-webhook
     template:
       metadata:
         labels:
           app: image-validation-webhook
       spec:
         containers:
         - name: webhook
           image: your-registry/webhook:v1
           ports:
           - containerPort: 443
             name: https
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: image-validation-webhook
   spec:
     ports:
     - port: 443
       targetPort: 443
     selector:
       app: image-validation-webhook
   ```

#### Step 3: Configure OpenShift to Use the Webhook

1. **Create a `ValidatingWebhookConfiguration`**:

   ```yaml
   apiVersion: admissionregistration.k8s.io/v1
   kind: ValidatingWebhookConfiguration
   metadata:
     name: image-policy-webhook
   webhooks:
   - name: image.validation.webhook
     clientConfig:
       service:
         name: image-validation-webhook
         namespace: your-namespace
         path: "/validate"
       caBundle: <base64-encoded-CA-cert>
     rules:
     - operations: ["CREATE", "UPDATE"]
       apiGroups: [""]
       apiVersions: ["v1"]
       resources: ["pods"]
     admissionReviewVersions: ["v1", "v1beta1"]
     sideEffects: None
     timeoutSeconds: 5
   ```

2. **Apply the `ValidatingWebhookConfiguration`**:

   ```bash
   oc apply -f validating-webhook.yaml
   ```

### Additional Considerations

- **TLS Configuration**: Ensure your webhook server is configured with TLS. You can generate a certificate and key and add them to your deployment.
- **Service Account Permissions**: Ensure the webhook service account has the necessary permissions to validate requests.
- **CA Bundle**: The CA bundle should be the certificate authority that signed the webhook's serving certificate. You can retrieve this from your cluster or use OpenShift's built-in CA.

This example demonstrates how to create a custom validation webhook in OpenShift to enforce policies around image usage. You can extend this webhook to check for additional image properties, labels, or any other criteria as needed.
