

This is the first iteration, following Scott's .NET framework, it will change once I look at implmenentation side of Scott's framework.
It gives high level idea of the structure. 
Azure Identity is the most vague/unknown part, as it currently doesn't exist in any language (.NET in progress).
Azure Identity structure is expected to change.

# Java
## Azure Core/Common
### Credentials Classes

The current classes in Azure core are not correctly implemented in the way:
1. The sync `ServiceClientCredentials` should not exist. 
2. The async `AsyncServiceClientCredentials` doesn't provide the `resource` or `scopes` on the interface method. The credential is not able to get the resource reliably from the `HttpRequest` object passed in.

### Authentication Policies

They are abandoned too because they only work with the credentials classes in azure core today.

## Azure Identity
The azure com.azure.identity package provides implementations for Active Directory based OAuth token authentication.  It provides an abstraction above the authentication libraries provided by the Azure Identity team.  It also provides mechanisms for querying credential information from the environment.

### Credential Implementations
~~~ java
package com.azure.identity;

    public abstract class TokenCredential {
        public abstract Mono<String> getTokenAsync(String resource);
    }

    // Use Fluent Pattern During Implementation
    //This is the package with most implementation needed.
    // Scott had the methods Static in his framework -- check the need and usage for it.
    public abstract class AzureCredential extends TokenCredential {
        public static final AzureCredential DEFAULT; //
        public List<HttpPipelinePolicy> createDefaultPipelinePolicies();
    }

    public class ClientSecretCredential extends AzureCredential {
        public ClientSecretCredential(String clientId, String clientSecret, String tenant);
        public ClientSecretCredential(String clientId, String clientSecret, String tenant, String aadEndpoint);

        @Override
        public Mono<String> getTokenAsync(String resource);
    }

    //Not in focus for now.
    public class ClientCertificateCredential extends AzureCredential {
        public ClientCertificateCredential(X509Certificate certificate, string authority, List<HttpPipelinePolicy> policies);
    }    

~~~
#### Questions / Open Issues:
- Should authority be required parameters?
- Possibly it's odd that these credential classes expose pipeline options when they're not "client" classes
    - Should we try to reuse the policies from the client class rather than having authentication calls have they're own pipeline.  Allowing us to avoid exposing options on the client (at least for now)   

### Credential Providers
~~~ java
package com.azure.identity;

    //Use Fluent Pattern during implementation
    public class TokenCredentialProvider {
        private final List<TokenCredentialProvider> providers;

        protected TokenCredentialProvider();
        public TokenCredentialProvider(List<TokenCredentialProvider> providers);
        public TokenCredentialProvider(TokenCredentialProvider... providers);

        public TokenCredential getCredential() throws CredentialNotFoundException;
        public Mono<String> getTokenAsync(String resource) throws CredentialNotFoundException;
    }

    public class EnvironmentCredentialProvider extends TokenCredentialProvider {
        public EnvironmentCredentialProvider();

        @Override
        public TokenCredential getCredential();
    }
    
    //
    public class MsiCredentialProvider extends TokenCredentialProvider {
        public MsiCredentialProvider(List<HttpPipelinePolicy> policies);

        // if available return cred
        protected Mono<TokenCredential> getCredential() throws CredentialNotFoundException;  // full implementation
    }    
~~~

### Client Library Implementations
~~~ java
package com.azure.keyvault


    //This exists in Secrets API, accepts AsyncServiceClientCredentials for async client, ServiceClientCredentials for sync client
     public final class SecretClientBuilder {
        private final List<HttpPipelinePolicy> policies;
        private TokenCredential credential;
        private HttpPipeline pipeline;
        private URL vaultEndpoint;
        private HttpClient httpClient;
        private HttpLogDetailLevel httpLogDetailLevel;
        private RetryPolicy retryPolicy;

        SecretClientBuilder() {
            retryPolicy = new RetryPolicy();
            httpLogDetailLevel = HttpLogDetailLevel.NONE;
            policies = new ArrayList<>();
        }
        public SecretClient build() {

            if (vaultEndpoint == null) {
                throw new IllegalStateException(KeyVaultErrorCodeStrings.getErrorString(KeyVaultErrorCodeStrings.VAULT_END_POINT_REQUIRED));
            }

            if (pipeline != null) {
                return new SecretAsyncClient(vaultEndpoint, pipeline);
            }

            if (credentials == null) {
                throw new IllegalStateException(KeyVaultErrorCodeStrings.getErrorString(KeyVaultErrorCodeStrings.CREDENTIALS_REQUIRED));
            }
            // Closest to API goes first, closest to wire goes last.
            final List<HttpPipelinePolicy> policies = new ArrayList<>();
            policies.add(new UserAgentPolicy(AzureKeyVaultConfiguration.SDK_NAME, AzureKeyVaultConfiguration.SDK_VERSION));
            policies.add(retryPolicy);
            policies.add(new KeyVaultCredentialPolicy(credentials));
            policies.addAll(this.policies);
            policies.add(new HttpLoggingPolicy(httpLogDetailLevel));

            HttpPipeline pipeline = httpClient == null
                ? new HttpPipeline(policies)
                : new HttpPipeline(httpClient, policies);

            return new SecretAsyncClient(vaultEndpoint, pipeline);
        }
    }
~~~

### End User Expected Usage Samples

~~~ java

  //Using ClientSecretCredential
  //Use Fluent Pattern for SecretCredentials or Providers.
    SecretClient client = new SecretClientBuilder()
        .vaultEndpoint("https://myvault.vault.azure.net")
        .credentials(AzureCredential.DEFAULT)
        .build();

~~~
