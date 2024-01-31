# Custom Keycloak Theme Deployment on AWS EKS using Helm Chart

This solves the issue of including a custom keycloak theme using only helm chart overrides. I could not find a simple solution online that did not use a custom Docker container, which is less flexible + maintainable than simply editing `overrides.yaml`.

This guide walks you through the process of deploying a custom Keycloak theme to an EKS cluster using Bitnami's Helm chart with the theme files hosted on an AWS S3 bucket.

## Prerequisites

- AWS CLI installed and configured
- kubectl installed and configured to interact with your EKS cluster
- Helm 3 installed

## Overview

The solution involves uploading the theme to an S3 bucket, creating an IAM policy and role for Kubernetes service accounts, and configuring the Helm chart to deploy Keycloak with the custom theme.

## Steps

### 1. Upload the Theme to S3

Begin by following [Keycloak's instructions](https://www.keycloak.org/docs/latest/server_development/index.html#_themes) for creating a custom theme. Public themes are available [here](https://github.com/topics/keycloak-theme). My file structure follows:

```plaintext
.
└── <theme-name>
    └── login
        ├── login.ftl
        ├── resources
        │   ├── css
        │   │   └── styles.css
        │   └── images
        │       └── Logo.png
        └── theme.properties
```

#### Using AWS CLI

1. Prepare your theme files locally in the structure required by Keycloak.
2. Create an S3 bucket or use an existing one:
   ```bash
   aws s3 mb s3://<your-bucket-name>
   ```
3. Upload your theme to the S3 bucket:
   ```bash
   aws s3 cp <local-theme-path> s3://<your-bucket-name>/<theme-path> --recursive
   ```

#### Using AWS Management Console

1. Log in to the AWS Management Console and navigate to the Amazon S3 service.
2. To create a new bucket, click on Create bucket:
    - Provide a unique bucket name.
    - Select the AWS Region closest to your user base for lower latency.
    - Click on Create at the bottom of the page.
3. Once the bucket is created, click on the bucket name to open it.
4. Click on Upload to open the upload dialog:
    - Click on Add files or Add folder to select your theme files or the entire theme folder respectively.
    - Adjust the permissions if necessary, so that the Keycloak service can access these files.
    - Click on Upload to transfer your theme to S3.

#### Important Considerations

- Ensure that your S3 bucket has the appropriate permissions set so that your Keycloak service can access the theme files. If the bucket or objects are not public, you will need to manage access via IAM policies or pre-signed URLs.
- For production environments, it is recommended to enable versioning on your S3 bucket to manage updates to your theme files effectively.
- Remember to replace placeholder values with actual values relevant to your AWS account and Keycloak theme.

### 2. Create an IAM Policy and Role

To allow the Keycloak pods to access the S3 bucket where your theme is stored, you need to create an IAM policy with the necessary permissions and an IAM role with the correct trust relationship that the Kubernetes service account can assume.

#### Creating an IAM Policy

1. Navigate to the IAM console within the AWS Management Console.
2. Click on "Policies" in the sidebar, then click the "Create policy" button.
3. Go to the JSON tab and enter a policy that allows access to your S3 bucket. Here is a sample policy:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::<your-bucket-name>",
                    "arn:aws:s3:::<your-bucket-name>/*"
                ]
            }
        ]
    }
    ```

4. Click "Review policy", give it a name and description, and then click "Create policy".
5. Note down the policy ARN after creation.

#### Creating an IAM Role with Trust Relationship

1. In the IAM console, click on "Roles" in the sidebar, then click the "Create role" button.
2. Select "EKS" from the list of services and then "EKS - Pod Identity" for the use case.
3. Click "Next: Permissions" and attach the policy you created earlier by searching for its name.
4. Click "Next: Tags" if you wish to add tags, otherwise just click "Next: Review".
5. Give your role a meaningful name and description.
6. Before you click "Create role", you need to establish a trust relationship between the role and your Kubernetes service account by editing the trust relationship. Click "Edit trust relationship" and use the following policy document:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>"
                },
                "Action": "sts:AssumeRoleWithWebIdentity",
                "Condition": {
                    "StringEquals": {
                        "oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
                    }
                }
            }
        ]
    }
    ```

7. After you've added the trust relationship and created the role, ensure the Kubernetes service account used by your Keycloak deployment is annotated with the new role's ARN. This step is crucial for the init container to have the permissions to access the S3 bucket. (Next section)


### 3. Configure Helm Chart Values

Use the `overrides.yaml` file to specify the necessary values for the init container and the theme volume. The configuration will instruct the Helm chart to:

- Add an init container to the Keycloak pods that downloads the theme from the S3 bucket.
- Mount the downloaded theme into the Keycloak pod.

```yaml
# Custom Keycloak theme configuration
extraVolumes:
  - name: theme-volume
    emptyDir: {}

extraVolumeMounts:
  - name: theme-volume
    mountPath: /opt/bitnami/keycloak/themes

initContainers:
  - name: init-keycloak-custom-theme
    image: amazon/aws-cli
    command: ["sh", "-c"]
    args:
      - >
        aws s3 cp s3://custom-keycloak-login-theme/themes /themes --recursive
    volumeMounts:
      - name: theme-volume
        mountPath: /themes

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/EKSInitContainerCustomKeycloakTheme
```

- **extraVolumes**: Defines a temporary storage volume named `theme-volume` using `emptyDir`, which will be used to store the custom Keycloak theme temporarily.
- **extraVolumeMounts**: Mounts the `theme-volume` to the Keycloak container at `/opt/bitnami/keycloak/themes`. This is where Keycloak expects its themes to be located.
- **initContainers**: Specifies an init container that uses the AWS CLI to download the custom theme from an S3 bucket (`s3://custom-keycloak-login-theme/themes`) to the theme-volume mounted at `/themes` within the init container. This container runs before the main Keycloak container starts, ensuring the theme is in place.
    - This configuration downloads your custom theme directly into a volume that's shared with the Keycloak container. This approach eliminates the need for a custom Docker image and allows for dynamic theme updates.
- **serviceAccount.annotations**: Adds an annotation to the service account to use an IAM role (`arn:aws:iam::<account-id>:role/EKSInitContainerCustomKeycloakTheme`). This IAM role should have the necessary permissions to access the S3 bucket.

The templated yaml exists within `helm.yaml` under `keycloak/templates/serviceaccount.yaml` and `keycloak/templates/statefulset.yaml`.

### 4. Deploy Keycloak using Helm

Deploy Keycloak with the custom theme using the Bitnami Helm chart:

- If you haven't already created the `keycloak` namespace, you can do so by running `kubectl create namespace keycloak` before deploying the Helm chart. This ensures your Keycloak deployment is organized within a specific namespace in your cluster.


```bash
helm upgrade --install --values overrides.yaml --namespace keycloak keycloak .
```

Monitor the deployment:

```bash
kubectl get pods
kubectl logs <keycloak-pod-name> -c init-keycloak-custom-theme
```

I recommend [k9s](https://k9scli.io/) as a monitoring alternative.

### Additional Notes

- Test the entire flow in a non-production environment before deploying to production.
- Consider the security implications of the permissions granted and follow the principle of least privilege.
- For a more detailed explanation, refer to the helm.yaml which is generated from the helm template command and includes the final rendered Kubernetes resources.
- The primary limitation of this solution is its specificity to AWS.
    - To adapt this approach for a different cloud environment or an on-premises Kubernetes setup, you would need to adjust the storage and access control mechanisms. Instead of an S3 bucket, you could utilize a manually-created ConfigMap for smaller themes or an equivalent object storage service offered by your cloud provider (e.g., Azure Blob Storage for Azure or Google Cloud Storage for GCP). Furthermore, the IAM roles utilized for granting permissions in AWS would be replaced by corresponding access control entities, such as Azure Role-Based Access Control (RBAC) or Google Cloud IAM policies, to grant the necessary permissions for your Kubernetes service accounts.
