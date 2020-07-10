# Bob's Books

Bob's Books is an example application made up of several components:

* which I will
* describe here

## Deploying the example application

To deploy the Bob's Books example application to an existing Verrazzano environment,
note that all of the commands shown must be executed against the Verrazzano management
cluster if you are using a multi-cluster environment.:

1. Create a `docker-registry` secret to enable pulling images from Oracle Container
   Registry.  This is needed to pull WebLogic and Coherence images.  Note that you
   may have already created this secret when installing Verrazzano itself.

   ```
   kubectl create secret docker-registry ocr \
           --docker-server=container-registry.oracle.com \
           --docker-username=YOUR_USERNAME \
           --docker-password=YOUR_PASSWORD \
           --docker-email=YOUR_EMAIL
   ```

   Replace `YOUR_USERNAME`, `YOUR_PASSWORD` and `YOUR_EMAIL` with the values that you
   use to access Oracle Container Registry.

1. Accept the license agreements for the WebLogic and Coherence images.  If you have not
   done so already, you must visit [Oracle Container Registry](https://container-registry.oracle.com)
   in your browser, navigate to the "Middleware" category and review and accept the
   licenses for the WebLogic and Coherence images.  You will not be able to pull these
   images until you have accepted the licenses.

1. Create a `docker-registry` secret to enable pulling images from GitHub Packages.
   **NOTE** This is a temporary requirement and will disappear when we go live!

   ```
   kubectl create secret docker-registry github-packages \
           --docker-server=docker.pkg.github.com \
           --docker-eusername=YOUR_GITHUB_USERNAME \
           --docker-password=YOUR_GITHUB_PERSONAL_ACCESS_TOEKSN \
           --docker-email=YOUR_EMAIL
   ```

   Replace `YOUR_GITHUB_USERNAME`, `YOUR_GITHUB_PERSONAL_ACCESS_TOKEN` and `YOUR_EMAIL` with
   the values that you use to access GitHub.

1. Create secrets containing the WebLogic administration credentials for the
   two domains:

   ```
   kubectl create secret generic bobs-bookstore-weblogic-credentials \
           --from-literal=username=weblogic \
           --from-literal=password=welcome1

   kubectl create secret generic bobbys-front-end-weblogic-credentials \
           --from-literal=username=weblogic \
           --from-literal=password=welcome1
   ```

   Note that the example applications are pre-configured to use these credentials.
   If you want to use different credentials, you will need to rebuild the
   Docker images for the example application.  The source code for the example
   applications is in the [examples repository](https://github.com/verrazzano/examples).

1. Create the secret containing the MySQL credentials:

   ```
   kubectl create secret generic mysql-credentials \
           --from-literal=username=books \
           --from-literal=password=WebLogic1234
   ```

1. Deploy the MySQL database:

   ```
   kubectl apply -f mysql.yaml
   ```

1. Deploy the Verrazzano Model for Bob's Books:

   ```
   kubectl apply -f bobs-books-model.yaml
   ```

1. Update the Verrazzano Binding for Bob's Books with correct DNS names for
   each of the applications.  This step is optional, you can use the IP
   address to access the applications if you do not have DNS names.

   The provided binding includes the following ingress bindings which use
   `*` as the DNS name.  This will result in Istio Virtual Services that
   do not require a DNS name.  It is recommended that you use real DNS
   names for the applications.  For example, if your zone is called
   `example.com`, you might create DNS A records called `bobbys-books.example.com`,
   `roberts-books.example.com` and `bobs-books.example.com`.

   Update the section shown below to sepcify the correct DNS names.  Each
   of the DNS A records should point to the external IP address of the
   `istio-ingressgateway` service in the `istio-system` namespace.

    ```
    ingressBindings:
    - name: "bobbys-ingress"
      dnsName: "*"
    - name: "bobs-ingress"
      dnsName: "*"
    - name: "roberts-ingress"
      dnsName: "*"
    ```


1. Deploy the Verrazzano Binding for Bob's Books:

   ```
   kubectl apply -f bobs-books-binding.yaml
   ```

## Explore the Bob's Books example application

TBD