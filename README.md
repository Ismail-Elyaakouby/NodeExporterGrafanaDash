How to use Vault to store secrets and use them in Jenkins | LaptrinhX{"@context":"https:\/\/schema.org","@type":"Article","publisher":{"@type":"Organization","name":"LaptrinhX","logo":"https:\/\/laptrinhx.com\/cdn\/favicon.ico"},"author":[{"@type":"Person","name":"bruj0","url":"https:\/\/laptrinhx.com\/author\/bruj0\/"}],"headline":"How to use Vault to store secrets and use them in Jenkins","url":"https:\/\/laptrinhx.com\/how-to-use-vault-to-store-secrets-and-use-them-in-jenkins-127703660\/","datePublished":"2020-05-03T16:50:11+07:00","keywords":"github, vault, vault client, vault ui, vault consul, docker, docker compose, jenkins, jenkins pipeline, secret management","description":"bruj0\/vault_jenkins How to use Vault to store secrets and use them in Jenkins Users starred: 93Users forked: 21Users watching: 93Updated at: 2020-05-03 16:50:11 How to...","mainEntityOfPage":{"@type":"WebPage","@id":"https:\/\/laptrinhx.com"},"image":"https:\/\/github.com\/bruj0\/vault_jenkins\/raw\/master\/image_0.png"}

[LaptrinhX](https://laptrinhx.com/)

- [My](#)
- [Tag](https://laptrinhx.com/tags/)
- [Author](https://laptrinhx.com/authors/)
- 
[Ebook](https://laptrinhx.com/tag/ebooks/)

- 
[Theme](https://laptrinhx.com/tag/theme-templates/)

- 
[Tutorial](https://laptrinhx.com/tag/tutorials/)

- 
[Funny](https://laptrinhx.com/tag/funny-vl/)

- 
[IT Job](https://laptrinhx.com/tag/it-jobs/)

- 
[Video](https://laptrinhx.com/tag/videos-ltx/)

- [#](#)

[https://laptrinhx.com/search/](https://laptrinhx.com/search/)[https://laptrinhx.com/archive/](https://laptrinhx.com/archive/)[https://feedly.com/i/subscription/feed/https://laptrinhx.com/rss/](https://feedly.com/i/subscription/feed/https://laptrinhx.com/rss/)[https://uptime.laptrinhx.com/](https://uptime.laptrinhx.com/)[https://laptrinhx.com/](https://laptrinhx.com/)[#](#)

[Search Post](Search Post)

- [Tools](https://laptrinhx.com/page/tools/)
- [Hacker News](https://laptrinhx.com/hackernews/)

[3 May 2020](https://laptrinhx.com/archive/2020-05-03/)/ [github](https://laptrinhx.com/tag/github/)/ 8 min read
# How to use Vault to store secrets and use them in Jenkins
![How to use Vault to store secrets and use them in Jenkins](https://github.com/bruj0/vault_jenkins/raw/master/image_0.png)
(adsbygoogle = window.adsbygoogle || []).push({});

### [bruj0/vault_jenkins](https://laptrinhx.com/link/?l=https%3A%2F%2Fgithub.com%2Fbruj0%2Fvault_jenkins)

How to use Vault to store secrets and use them in Jenkins

- Users starred: 93
- Users forked: 21
- Users watching: 93
- Updated at: 2020-05-03 16:50:11

---

# [#how-to-use-hashicorp-vault-to-store-secrets-and-read-them-from-jenkins](#how-to-use-hashicorp-vault-to-store-secrets-and-read-them-from-jenkins)How to use HashiCorp Vault to store secrets and read them from Jenkins

by Rodrigo A. Diaz Leven

- [How to use HashiCorp Vault to store secrets and read them from Jenkins](#how-to-use-hashicorp-vault-to-store-secrets-and-read-them-from-jenkins)
- [Description](#description)
- [Why do we want to use it.](#why-do-we-want-to-use-it)
- [The setup](#the-setup)

- [Initial Vault configuration](#initial-vault-configuration)
- [Checking Vault is unsealed](#checking-vault-is-unsealed)
- [Testing](#testing)

- [Reading secrets from Jenkins](#reading-secrets-from-jenkins)
- [AppRole](#approle)
- [Generating Policies and Roles](#generating-policies-and-roles)
- [Generating Role ID](#generating-role-id)
- [Generating Jenkins Token ID](#generating-jenkins-token-id)
- [Configuring Jenkins credentials to access Vault](#configuring-jenkins-credentials-to-access-vault)
- [Reading the secret](#reading-the-secret)
- [Retrieving secrets from source code](#retrieving-secrets-from-source-code)
- [Job Output](#job-output)

- [Optional: Installing Vault plugin for Jenkins](#optional--installing-vault-plugin-for-jenkins)
- [References](#references)

## [#description](#description)Description

In this article we will learn how to store secret or any other type of information you wish like Certificates in [Vault](https://laptrinhx.com/link/?l=https%3A%2F%2Fwww.vaultproject.io%2F) .

Vault works by exchanging **information** for **secrets**, for example a client could havea RoleID and a SecretID or a temporary Token that it can trade for the actual Secret.

It also can create temporary access to a 3rd party services like AWS through the use of back ends.

## [#why-do-we-want-to-use-it](#why-do-we-want-to-use-it)Why do we want to use it.

- 
Centralized secrets storage and role based access.

- 
Replace secrets stored in source code or shell scripts which then are stored in places, difficult to replace with one time or temporary authentication credentials.

- 
Create dynamic authentication for short lived jobs, ie AWS infrastructure creation. This means that a short lived set of AWS secret/key pair is created for a job and they won't need to be hard coded.

- 
Optional, Human authentication to services like SSH, Jenkins, VPN, etc.

## [#the-setup](#the-setup)The setup

We will test this by using a docker-compose project that will orchestrate 3 containers:

- Consul: service discovery and backend for storing encrypted secrets
- Vault: Service for converting tokens to secrets
- Jenkins: Automation

We persist Jenkins configuration in a Docker volume called "jenkins-data" and bind the local directory config to the Consul container so we can copy configuration later.

    version: '2'services:
     consul:
       image: "consul"hostname: "consul"command: "agent -dev -client 0.0.0.0"ports:
         - "8400:8400"
         - "8500:8500"
         - "8600:53/udp"vault:
       depends_on:
         - consulimage: "vault"hostname: "vault"links:
         - "consul:consul"environment:
         VAULT_ADDR: http://127.0.0.1:8200ports:
         - "8200:8200"volumes:
         - ./tools/wait-for-it.sh:/wait-for-it.sh
         - ./config/vault:/config
         - ./config/vault/policies:/policiesentrypoint: /wait-for-it.sh -t 20 -h consul -p 8500 -s -- vault server -config=/config/with-consul.hclblueocean:
       depends_on:
         - vaultimage: "jenkinsci/blueocean:latest"ports:
         - "8080:8080"volumes:
         - /var/run/docker.sock:/var/run/docker.sock
         - 'jenkins-data:/var/jenkins_home'volumes:
     jenkins-data:
    

# [#initial-vault-configuration](#initial-vault-configuration)Initial Vault configuration

The first command starts the container and the second one logs us in the Vault container to "unseal" it.

This is the term used by Vault to say the service is ready to be used and to do so we need 3 of the 5 keys that were generated by the init command.

This is only needed the first time we setup vault and you should store the keys in a secure place.

    $ docker-compose up -d
    Creating vaultjenkins_consul_1 ... 
    Creating vaultjenkins_consul_1 ... done
    Creating vaultjenkins_vault_1 ... 
    Creating vaultjenkins_vault_1 ... done
    Creating vaultjenkins_blueocean_1 ... 
    Creating vaultjenkins_blueocean_1 ... done
    
    $ docker exec -it vaultjenkins_vault_1 sh
    
    (vaultjenkins_vault_1)$ vault init
    Unseal Key 1: d28dc3e20848c499749450b411bdc55416cefb0ff6ddefd01ec02088aa5c90aa01
    Unseal Key 2: ad2b7e9d02d0c1cb5b98fafbc2e3ea56bd4d4fa112a0c61882c1179d6c6585f302
    Unseal Key 3: c393269f177ba3d07b14dbf14e25a325205dfbf5c91769b8e55bf91aff693ce603
    Unseal Key 4: 87c605e5f766d2f76d39756b486cbdafbb1998e72d2f766c40911f1a288e53a404
    Unseal Key 5: e97e5de7e2cdb0ec4db55461c4aaf4dc26092cb3f698d9cc270bf19dbb82eab105
    Initial Root Token: 5a4a7e11-1e2f-6f76-170e-b8ec58cd2da5
    
    (vaultjenkins_vault_1)$ vault unseal (run this 3 times with 3 different keys from above)
    

## [#checking-vault-is-unsealed](#checking-vault-is-unsealed)Checking Vault is unsealed

If the unseal was successful then you can login to the UI by going to:

[http://localhost:8500/ui/#/dc1/services/vault](https://laptrinhx.com/link/?l=http%3A%2F%2Flocalhost%3A8500%2Fui%2F%23%2Fdc1%2Fservices%2Fvault)

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_0.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_0.png)

## [#testing](#testing)Testing

The first command sets an alias so we dont have to keep typing the same thing over and over.

(adsbygoogle = window.adsbygoogle || []).push({});

We will use the Vault client from inside the container but you can use one installed in your local machine too.

    $ alias vault='docker exec -it vaultjenkins_vault_1 vault "$@"'
    
    $ vault write -address=http://127.0.0.1:8200 secret/billion-dollars value=behind-super-secret-password
    
    $ vault read secret/billion-dollars
    Key             	Value
    ---             	-----
    refresh_interval	2592000
    value           	behind-super-secret-password

Here we store a secret text "behind-super-secret-password" in the path "secret/billion-dollars" and check in the UI if it was recorded encrypted.

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_1.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_1.png)

# [#reading-secrets-from-jenkins](#reading-secrets-from-jenkins)Reading secrets from Jenkins

## [#approle](#approle)AppRole

AppRole is a secure introduction method to establish machine identity.

In AppRole, in order for the application to get a token, it would need to login using a Role ID (which is **static**, and associated with a policy), and a Secret ID (which is **dynamic**, one time use, and can only be requested by a previously authenticated user/system. In this case, we have two options:

- 
Store the Role IDs in Jenkins

- 
Store the Role ID in the Jenkinsfile of each project

## [#generating-policies-and-roles](#generating-policies-and-roles)Generating Policies and Roles

Now Jenkins will need permissions to retrieve Secret IDs for our newly created role. Jenkins shouldn’t be able to access the secret itself, list other Secret IDs, or even the Role ID.

In this case, tokens assigned to the **java-example** policy would have permission to read a secret on the **secret/hello** path.

    $ echo'path "auth/approle/role/java-example/secret-id" {  capabilities = ["read","create","update"]}'> ./config/vault/policies/jenkins_policy.hcl
    
    $ vault policy-write jenkins ./config/vault/policies/jenkins_policy.hcl
    

Now we have to create a **Role **that will generate **tokens **associated with that **policy**, and retrieve the token:

    $ echo'path "secret/hello" {  capabilities = ["read", "list"]}'> ./config/vault/policies/java-example_policy.hcl
    $ vault policy-write java-example ./config/vault/policies/java-example_policy.hcl
    
    Policy 'java-example' written.

    $ vault auth-enable approle
    $ vault write auth/approle/role/java-example \
    > secret_id_ttl=60m \
    > token_ttl=60m \
    > token_max_tll=120m \
    > policies="java-example"
    Success! Data written to: auth/approle/role/java-example
    
    $ vault read auth/approle/role/java-example
    Key                 Value
    ---                 -----
    bind_secret_id      true
    bound_cidr_list
    period              0
    policies            [default java-example]
    secret_id_num_uses  0
    secret_id_ttl       3600
    token_max_ttl       0
    token_num_uses      0
    token_ttl           3600
    

## [#generating-role-id](#generating-role-id)Generating Role ID

We generate a **Role ID** for our application that can be used in conjunction with the Jenkins **Token ID** to generate a **Secret ID** that will allow us to access the secret at the path "secret/hello" per our policy.

    $ vault read auth/approle/role/java-example/role-id
    Key    	Value
    ---    	-----
    role_id	73ea4552-53da-6844-bab0-0d39d2dc06aa

## [#generating-jenkins-token-id](#generating-jenkins-token-id)Generating Jenkins Token ID

Jenkins needs a long lived token but it needs to be eventually rotated.

    $ vault token-create -policy=jenkins
    Key            	Value
    ---            	-----
    token          	c70b74af-0e46-40da-ae15-c09b95a636f9
    token_accessor 	1c1ad1bf-1b56-e398-169a-7e14600a3cb3
    token_duration 	768h0m0s
    token_renewable	true
    token_policies 	[default jenkins]

## [#configuring-jenkins-credentials-to-access-vault](#configuring-jenkins-credentials-to-access-vault)Configuring Jenkins credentials to access Vault

This is the long lived Token ID we generated for Jenkins use.

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_4.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_4.png)

This is the Role ID for our example app and is restricted to "secret/hello" by our policy, it can also be hardcoded in the source code or job script.

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_3.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_3.png)

## [#reading-the-secret](#reading-the-secret)Reading the secret

Let’s write a secret that our application will consume:

(adsbygoogle = window.adsbygoogle || []).push({});

    $ vault write secret/hello value="You've Successfully retrieved a secret from Hashicorp Vault"
    Success! Data written to: secret/hello
    

We are going to exchange **ROLE_ID **(belonging only to this particular job) and a temporary **SECRET_ID **(generated with the Jenkins token) for another **SECRET_ID**.

This second **SECRET_ID** has a policy attached that will only let us access the secrets at the path "secret/hello".

This way we can authenticate our jobs and assign them specific access to our secrets, without leaving any secrets in our our source code or shell scripts.

    pipeline {
      agent any
    stages {  
      stage('Integration Tests') {
          steps {
          sh 'curl -s -o vault.zip https://releases.hashicorp.com/vault/0.9.1/vault_0.9.1_linux_amd64.zip ; yes | unzip vault.zip '
            withCredentials([string(credentialsId: 'JAVA_EXAMPLE_ROLE_ID', variable: 'ROLE_ID'),string(credentialsId: 'JENKINS_VAULT_TOKEN', variable: 'VAULT_TOKEN')]) {
            sh '''  set +x  export VAULT_ADDR=http://vault:8200  export VAULT_SKIP_VERIFY=true  export SECRET_ID=$(./vault write -field=secret_id -f auth/approle/role/java-example/secret-id)  export JOB_VAULT_TOKEN=$(./vault write -field=token auth/approle/login role_id=${ROLE_ID}   secret_id=${SECRET_ID})./vault read -address=http://vault:8200 secret/hello '''
        }
       }
      }
     }
    }

- 
We retrieve a **SECRET_ID **A using Jenkins administrative **JENKINS_VAULT_TOKEN **(that we’ve manually generated before). That’s the only one that would be relatively longed lived, but can only generate SECRET_IDs. Notice that we store this SECRET_ID A in the environment variable called VAULT_TOKEN so the vault client can use it.

- 
    ./vault write -field=secret_id -f auth/approle/role/java-example/secret-id
    

​

- 
We do an AppRole login with the **ROLE_ID **and the **SECRET_ID A **(short lived), this gives us a second **SECRET_ID B** that we store in the environment variable VAULT_TOKEN:

- 
    ./vault write -field=token auth/approle/login role_id=${ROLE_ID} secret_id=${SECRET_ID}
    

- 
We use this **SECRET_ID B** to read our plain text secret:

- 
    ./vault read -address=http://vault:8200 secret/hello 
    

​

## [#retrieving-secrets-from-source-code](#retrieving-secrets-from-source-code)Retrieving secrets from source code

We can also retrieve the secrets from an application at run time, using environment variables to authenticate to Vault like this:

    <td>String vaulttoken =System.getenv("VAULT_TOKEN");
        String vaulthost =System.getenv("VAULT_ADDR");
        System.out.format( "Using Vault Host %s\n", vaulthost);
        System.out.format( "With Vault Token %s\n", vaulttoken );
        /* This should be a separate method called from Main, then     * again for simplicity...*/finalVaultConfig config =newVaultConfig().build();
        finalVault vault =newVault(config);
        try {
        finalString value = vault.logical()
                       .read("secret/hello")
                       .getData().get("value");
        System.out.format( "value key in secret/hello is "+ value +"\n");
        } catch(VaultException e) {
          System.out.println("Exception thrown: "+ e);
        }

## [#job-output](#job-output)Job Output

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_5.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_5.png)

# [#optional-installing-vault-plugin-for-jenkins](#optional-installing-vault-plugin-for-jenkins)Optional: Installing Vault plugin for Jenkins

More information: [https://github.com/jenkinsci/hashicorp-vault-plugin](https://laptrinhx.com/link/?l=https%3A%2F%2Fgithub.com%2Fjenkinsci%2Fhashicorp-vault-plugin)

[![image alt text](https://github.com/bruj0/vault_jenkins/raw/master/image_2.png)](https://github.com/bruj0/vault_jenkins/blob/master/image_2.png)

If we decide to use this plugin then we will be using the "[Response Wrapping](https://laptrinhx.com/link/?l=https%3A%2F%2Fwww.vaultproject.io%2Fdocs%2Fconcepts%2Fresponse-wrapping.html)" feature from vault.

This feature creates one time use tokens that are used to access specific secrets and limiting the time they can be used.

    node {
      // define the secrets and the env variablesdef secrets = [
          [$class: 'VaultSecret', path: 'secret/testing', secretValues: [
              [$class: 'VaultSecretValue', envVar: 'testing', vaultKey: 'value_one'],
              [$class: 'VaultSecretValue', envVar: 'testing_again', vaultKey: 'value_two']]],
          [$class: 'VaultSecret', path: 'secret/another_test', secretValues: [
              [$class: 'VaultSecretValue', envVar: 'another_test', vaultKey: 'value']]]
      ]
    
      // optional configuration, if you do not provide this the next higher configuration// (e.g. folder or global) will be useddef configuration = [$class: 'VaultConfiguration',
                           vaultUrl: 'http://my-very-other-vault-url.com',
                           vaultCredentialId: 'my-vault-cred-id']
    
      // inside this block your credentials will be available as env variables
      wrap([$class: 'VaultBuildWrapper', configuration: configuration, vaultSecrets: secrets]) {
          sh 'echo $testing'
          sh 'echo $testing_again'
          sh 'echo $another_test'
      }
    }

# [#references](#references)References

A lot of information for this article was used from:

- [http://nicolas.corrarello.com/general/vault/security/ci/2017/04/23/Reading-Vault-Secrets-in-your-Jenkins-pipeline.html](https://laptrinhx.com/link/?l=http%3A%2F%2Fnicolas.corrarello.com%2Fgeneral%2Fvault%2Fsecurity%2Fci%2F2017%2F04%2F23%2FReading-Vault-Secrets-in-your-Jenkins-pipeline.html)

The Docker compose project was based on:

- 
[https://github.com/tolitius/cault](https://laptrinhx.com/link/?l=https%3A%2F%2Fgithub.com%2Ftolitius%2Fcault)

​

---

[github](https://laptrinhx.com/tag/github/)[vault](https://laptrinhx.com/tag/vault/)[vault client](https://laptrinhx.com/tag/vault-client/)[vault ui](https://laptrinhx.com/tag/vault-ui/)[vault consul](https://laptrinhx.com/tag/vault-consul/)[docker](https://laptrinhx.com/tag/docker/)[docker compose](https://laptrinhx.com/tag/docker-compose/)[jenkins](https://laptrinhx.com/tag/jenkins/)[jenkins pipeline](https://laptrinhx.com/tag/jenkins-pipeline/)[secret management](https://laptrinhx.com/tag/secret-management/)

(adsbygoogle = window.adsbygoogle || []).push({});
B
#### [bruj0](https://laptrinhx.com/author/bruj0/)

Read [more posts](https://laptrinhx.com/author/bruj0/) by this author.

[Read More](https://laptrinhx.com/author/bruj0/)

[Latest Posts](https://laptrinhx.com)


const RELATED_TAG  = 'github', RELATED_ID   = '127703660', PUBLISHED_AT = '1588499411';

&mdash; LaptrinhX &mdash;
### [github](https://laptrinhx.com/tag/github/)
[ →](https://laptrinhx.com/tag/github/)

[![LaptrinhX icon](https://laptrinhx.com/cdn/favicon.png)LaptrinhX](https://laptrinhx.com)
&mdash;
How to use Vault to store secrets and use them in Jenkins

Share this 
[https://twitter.com/share?text=How+to+use+Vault+to+store+secrets+and+use+them+in+Jenkins&amp;url=https://laptrinhx.com/how-to-use-vault-to-store-secrets-and-use-them-in-jenkins-127703660/](https://twitter.com/share?text=How+to+use+Vault+to+store+secrets+and+use+them+in+Jenkins&amp;url=https://laptrinhx.com/how-to-use-vault-to-store-secrets-and-use-them-in-jenkins-127703660/)[https://www.facebook.com/sharer/sharer.php?u=https://laptrinhx.com/how-to-use-vault-to-store-secrets-and-use-them-in-jenkins-127703660/](https://www.facebook.com/sharer/sharer.php?u=https://laptrinhx.com/how-to-use-vault-to-store-secrets-and-use-them-in-jenkins-127703660/)

[LaptrinhX](https://laptrinhx.com) &copy; 2020
[https://www.facebook.com/LaptrinhX-1909354179164106/](https://www.facebook.com/LaptrinhX-1909354179164106/)
[Latest Posts](https://laptrinhx.com)[bdev.dev](https://bdev.dev)[raoxyz](https://raoxyz.com)[congtyaz](https://congtyaz.com)
