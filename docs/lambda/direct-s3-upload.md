# Direct S3 Upload (CLI & Console)

Use this guide when the Lambda image service is unavailable, too expensive for one-off uploads, or you need to upload static assets (logos, icons, documents) that don't go through the admin panel.

---

## When to use this vs Lambda

| Situation | Method |
| --------- | ------ |
| Admin panel image upload (end users) | Lambda pre-signed URL |
| One-off assets — logo, favicon, static files | AWS CLI or Console |
| Bulk / batch migration of existing images | AWS CLI `sync` |
| Budget constraint — Lambda cold-starts or high invocation volume | AWS CLI or Console |
| No internet access to Lambda endpoint | AWS CLI |

---

## Prerequisites

=== "AWS CLI"

    1. Install the AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
    2. Configure the `tensor` profile (one-time):

        ```bash
        aws configure --profile tensor
        ```

        Enter when prompted:

        | Field | Value |
        | ----- | ----- |
        | AWS Access Key ID | *(from IAM user)* |
        | AWS Secret Access Key | *(from IAM user)* |
        | Default region | `eu-north-1` |
        | Default output format | `json` |

    3. Verify access:

        ```bash
        aws --profile tensor s3 ls s3://tensor-labz-store/
        ```

=== "AWS Console"

    1. Open the AWS Console: https://console.aws.amazon.com/s3
    2. Sign in with your IAM credentials
    3. Navigate to **S3 → tensor-labz-store**

---

## Upload a single file

=== "AWS CLI"

    ```bash
    aws --profile tensor s3 cp <local-file> s3://tensor-labz-store/<folder>/<filename> \
      --acl public-read \
      --content-type <mime-type>
    ```

    **Examples:**

    ```bash
    # Upload logo (PNG)
    aws --profile tensor s3 cp src/assets/images/logo.png \
      s3://tensor-labz-store/assets/upload/logo.png \
      --acl public-read \
      --content-type image/png

    # Upload dark mode logo
    aws --profile tensor s3 cp src/assets/images/logo-dark.png \
      s3://tensor-labz-store/assets/upload/logo-dark.png \
      --acl public-read \
      --content-type image/png

    # Upload a WebP image
    aws --profile tensor s3 cp banner.webp \
      s3://tensor-labz-store/Home/Hero/banner.webp \
      --acl public-read \
      --content-type image/webp
    ```

=== "AWS Console"

    1. Open **S3 → tensor-labz-store**
    2. Navigate to the target folder (e.g. `assets/upload/`) — create it if needed
    3. Click **Upload → Add files**
    4. Select your file
    5. Expand **Permissions** → enable **Grant public-read access**
    6. Click **Upload**

---

## Upload a folder (batch)

=== "AWS CLI"

    ```bash
    aws --profile tensor s3 sync ./local-folder \
      s3://tensor-labz-store/remote-folder/ \
      --acl public-read
    ```

    Example — sync all images in `dist/assets`:

    ```bash
    aws --profile tensor s3 sync ./dist/assets \
      s3://tensor-labz-store/assets/ \
      --acl public-read
    ```

=== "AWS Console"

    1. Navigate to the target folder in S3
    2. Click **Upload → Add folder**
    3. Select your local folder
    4. Enable **Grant public-read access**
    5. Click **Upload**

---

## Get the public URL

After upload, the public URL follows this pattern:

```
https://tensor-labz-store.s3.eu-north-1.amazonaws.com/<folder>/<filename>
```

**Examples:**

| File | Public URL |
| ---- | ---------- |
| `assets/upload/logo.png` | `https://tensor-labz-store.s3.eu-north-1.amazonaws.com/assets/upload/logo.png` |
| `assets/upload/logo-dark.png` | `https://tensor-labz-store.s3.eu-north-1.amazonaws.com/assets/upload/logo-dark.png` |
| `Home/Hero/banner.webp` | `https://tensor-labz-store.s3.eu-north-1.amazonaws.com/Home/Hero/banner.webp` |

=== "AWS CLI"

    You can also print the URL immediately after upload:

    ```bash
    FILE=logo.png
    FOLDER=assets/upload
    BUCKET=tensor-labz-store
    REGION=eu-north-1

    aws --profile tensor s3 cp $FILE s3://$BUCKET/$FOLDER/$FILE \
      --acl public-read \
      --content-type image/png

    echo "Public URL: https://$BUCKET.s3.$REGION.amazonaws.com/$FOLDER/$FILE"
    ```

=== "AWS Console"

    1. Click the uploaded file
    2. Copy the **Object URL** from the properties panel

---

## S3 folder structure

```
tensor-labz-store/
├── assets/
│   └── upload/          ← logos, favicons, static brand assets
├── Home/
│   └── Hero/            ← hero slide images
├── Insights/            ← service / insight images
├── projects/
│   └── {slug}/          ← project images (cover + extras)
└── about-us/            ← about page media
```

Always place files in the correct folder so URLs stay predictable.

---

## Delete a file

=== "AWS CLI"

    ```bash
    aws --profile tensor s3 rm s3://tensor-labz-store/<folder>/<filename>
    ```

=== "AWS Console"

    1. Navigate to the file in S3
    2. Select the checkbox next to it
    3. Click **Delete**

---

## Common MIME types

| Extension | `--content-type` |
| --------- | ---------------- |
| `.png`    | `image/png` |
| `.jpg` / `.jpeg` | `image/jpeg` |
| `.webp`   | `image/webp` |
| `.gif`    | `image/gif` |
| `.svg`    | `image/svg+xml` |
| `.pdf`    | `application/pdf` |
| `.mp4`    | `video/mp4` |

!!! warning "Never put credentials in the frontend"
    Do NOT add `VITE_AWS_ACCESS_KEY` or `VITE_AWS_SECRET` to `.env`.
    AWS credentials must only be used server-side (Lambda) or locally via the CLI.
    The frontend always uses pre-signed URLs from Lambda.
