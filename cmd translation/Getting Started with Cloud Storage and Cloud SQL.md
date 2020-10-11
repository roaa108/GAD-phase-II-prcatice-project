# Getting Started with Cloud Storage and Cloud SQL

https://googlepluralsight.qwiklabs.com/focuses/10910845?parent=lti_session



## Task 1: Deploy a web server VM instance

keep **Region** and **Zone**, assigned by Qwiklabs. default **Machine type**,**Image** : **Debian GNU/Linux 9 (stretch)**, **Allowed HTTP traffic**. With **Startup script**.

- create bucket for startup script

  ```sh
  export LOCATION=US
  ```

  ```sh
  gsutil mb -l $LOCATION gs://$DEVSHELL_PROJECT_ID
  ```

  ```sh
  gsutil cp Desktop/startup-script.sh gs://$DEVSHELL_PROJECT_ID
  ```

- create the vm instance

```sh
gcloud compute instances create bloghost --zone us-central1-a --image-project 'debian-cloud' --image "debian-9-stretch-v20190213" --subnet='default' --tags http --metadata startup-script-url=https:https://storage.cloud.google.com/qwiklabs-gcp-03-ed08883bd68f/startup-script.sh?authuser=0
```

- allow http traffic

```sh
gcloud compute --project=qwiklabs-gcp-03-ed08883bd68f firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```



## Task 2: Create a Cloud Storage bucket 

```sh
gsutil mb -l $LOCATION gs://$DEVSHELL_PROJECT_ID
```

- Retrieve a banner image from a publicly accessible Cloud Storage location:

```sh
gsutil cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png
```

- Copy the banner image to your newly created Cloud Storage bucket:

```sh
gsutil cp my-excellent-blog.png gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
```

- Modify the Access Control List of the object you just created so that it is readable by everyone:

```sh
gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
```



## Task 3: Create the Cloud SQL instance

```sh
gcloud sql instances create blog-db --tier=db-n1-standard-2 
--region=us-central1-a
```

```sh
gcloud sql users set-password root --host=% --instance blog-db --password 123456
```

```sh
gcloud sql users set-password blogdbuser --instance blog-db --password 123456
```



## Task 4: Configure an application in a Compute Engine instance to use Cloud SQL

```sh
ssh bloghost
```

```sh
cd /var/www/html
```

```sh
sudo nano index.php
```

paste this in the opening file

```html
<html>
<head><title>Welcome to my excellent blog</title></head>
<body>
<h1>Welcome to my excellent blog</h1>
<?php
 $dbserver = "CLOUDSQLIP";
$dbuser = "blogdbuser";
$dbpassword = "DBPASSWORD";
// In a production blog, we would not store the MySQL
// password in the document root. Instead, we would store it in a
// configuration file elsewhere on the web server VM instance.

$conn = new mysqli($dbserver, $dbuser, $dbpassword);

if (mysqli_connect_error()) {
        echo ("Database connection failed: " . mysqli_connect_error());
} else {
        echo ("Database connection succeeded.");
}
?>
</body></html>
```

- Press **Ctrl+O** then  **Enter** to save edited file, **Ctrl+X** to exit ,  then nano text Restart the web server

```sh
sudo service apache2 restart
```

- paste the ip address followed by **/index.php**

```sh
35.192.208.2/index.php
```



## Task 5: Configure an application in a Compute Engine instance to use a Cloud Storage object

- Return ssh session on **bloghost** VM instance.

```sh
ssh bloghost
```

- set  working directory to the document root of the web server:

```
cd /var/www/html
```

- edit **index.php**:

```
sudo nano index.php
```

- get the piblic link of the bucket object **my-excellent-blog.png** and paste it here

```sh
<img src=' [LINK] '>
```

- Ctrl+O , then  **Enter** to save edited file,  **Ctrl+X** to exit the text editor then Restart the web server:

```sh
sudo service apache2 restart
```

