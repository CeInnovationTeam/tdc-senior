# Lab. #1 - Deployment Automation

⏱ Tempo de execução: ~65min.

Neste lab, você construirá uma esteira de desenvolvimento com o serviço **OCI DevOps**, que irá automatizar a entrega da aplicação MuShop, de forma conteinerizada, a um cluster Kubernetes na OCI!

Para aprofundar seu conhecimento neste serviço, acesse os links abaixo! 👇

- 🌀 [Página oficial do OCI DevOps](https://www.oracle.com/devops/devops-service/)
- 🧾 [Documentação do OCI DevOps](https://docs.oracle.com/en-us/iaas/Content/devops/using/home.htm)

**Confira todo o passo-a-passo dessa implementação:**
- [Pre Reqs: Coletar dados necessários](#PreReqs)
- [Passo 1: Espelhar um repo no github para o projeto OCI DevOps](#Passo1)
  - [Passo 1.1: GitHub repo e Personal Access Token](#Passo1.1)
  - [Passo 1.2: Vault Secret](#Passo1.2)
  - [Passo 1.3: Notifications Topic](#Passo1.3)
  - [Passo 1.4: Criação de External Connection](#Passo1.4)
  - [Passo 1.5: Repo Github espelhado](#Passo1.5)
- [Passo 2: Criar pipeline de build e configurar criação da container image (CI)](#Passo2)
- [Passo 3: Configurar entrega da container image (CI)](#Passo3)
- [Passo 4: Criar pipeline de deploy e configurar entrega da aplicação ao cluster Kubernetes (CD)](#Passo4)
  - [Passo 4.1: Criação do kubernetes secret](#Passo4.1)
  - [Passo 4.2: Adição de environment no OCI DevOps](#Passo4.2)
  - [Passo 4.3: Criação de artefato de deployment](#Passo4.3)
- [Passo 5: Criação do trigger de início e do trigger de conexão dos pipelines de CI e de CD](#Passo5)
- [Passo 6: Validar implementação](#Passo6)

 - - -

 ## <a name="PreReqs"></a> Pre Reqs: Coletar dados necessários
 
Vamos coletar alguns dados na tenancy da OCI que serão utilizados ao longo do laboratório. Recomendamos que os anote em um bloco de notas para ter sempre em mãos, de modo fácil. Serão coletados os seguintes dados:

```bash
Tenancy Namespace:
Auth Token:
Código da Região:
```

### Tenancy Namespace

1. Faça o [login](https://www.oracle.com/cloud/sign-in.html) em sua conta na OCI. 

2. No menu do lado direto, no ícone do usuário, clique no nome da sua tenancy.

![](./Images/namespace1.png)

3. Agora copie o namespace para o bloco de notas.

![](./Images/namespace2.png)

### Username e Auth Token

1. No menu do lado direito, clique no ícone do usuário, e então no nome do seu usuário.

![](./Images/user1.png)

2. Copie o dado destacado em vermelho e o insira no seu bloco de notas. Este será o seu 'username'.

![](./Images/username.png)

3. Depois, desça a página até visualizar 'Resources', então clique em **Auth Tokens** e em **Generate Token**, para gerar um novo token.

![](./Images/user2.png)

4. Insira uma descrição.

![](./Images/token_description.png)

5. Salve o auth token gerado no bloco de notas.

![](./Images/generated_token.png)

### Código da Região
1. Para a região US East (Ashburn): 'iad'.

2. Para a região de Brazil East (Sao Paulo): 'gru'.

3. Para as demais regiões, confira neste [link](https://docs.oracle.com/en-us/iaas/Content/Registry/Concepts/registryprerequisites.htm#regional-availability).

2. Copie o valor correspondente à sua região para o bloco de notas.
 
É isso! Cumprimos todos os pré-requisitos para o laboratório! Vamos para os próximos passos!

- - -

## <a name="Passo1"></a> Passo 1: Espelhar um repo no github para o projeto OCI DevOps

### <a name="Passo1.1"></a> Passo 1.1: GitHub repo e Personal Access Token

1. Crie um repo no github.

![](./Images/mushop_repo.png)

2. Clique no ícone do seu perfil e em **Settings**.

![](./Images/github_PAT1.png)

3. No menu do lado esquerdo, clique em **Developer settings**.

![](./Images/github_PAT2.png)

4. Feito isto, clique em **Personal access tokens** e **Generate new token**.

![](./Images/github_PAT3.png)

5. Insira uma nota descritiva e selecione o tempo de expiração que deseja. Para o escopo, selecione **public_repo**.

![](./Images/github_PAT4.png)

6. Ao final da página, clique em **Generate Token**.

![](./Images/github_PAT5.png)

7. Copie o token gerado.

![](./Images/github_PAT6.png)

### <a name="Passo1.2"></a> Passo 1.2: Vault Secret
Nesse momento, vamos provisionar um **OCI Vault** para armazenar o token gerado no GitHub como um vault secret. O **OCI Vault** é um serviço da OCI que permite o gerenciamento seguro de credenciais e de outros dados sensíveis com chaves de criptografia.

Para aprofundar seu conhecimento neste serviço, acesse os links abaixo! 👇

- 🔑 [Página oficial do OCI Vault](https://www.oracle.com/security/cloud-security/key-management/)
- 🧾 [Documentação do OCI Vault](https://docs.oracle.com/en-us/iaas/Content/KeyManagement/home.htm)

1. Na OCI, no menu de hambúrguer 🍔, acesse: **Identity & Security** → **Vault**.

![](./Images/oci_vault1.png)

2. Clique em **Create Vault**.

![](./Images/oci_vault2.png)

3. Atribua um nome ao seu vault e clique em **Create Vault**.

![](./Images/oci_vault3.png)

4. Para criar uma Master Encryption Key, clique em **Create Key**.

![](./Images/oci_vault_create1.png)

5. Selecione o 'Protection Mode' como **Software**, atribua um nome à chave e clique em **Create Key**.

![](./Images/oci_vault_create2.png)

6. Feito isto, clique em **Secrets** e em **Create Secret**.

![](./Images/oci_vault_create3.png)

7. Atribua um nome ao secret, selecione a Master Encryption Key criada anteriormente, insira o Personal Access Token em 'Secrets Contents' e clique em **Create Secret**.

![](./Images/oci_vault_create4.png)

### <a name="Passo1.3"></a> Passo 1.3: Notifications Topic
Nessa etapa, vamos criar um tópico para que o projeto do OCI DevOps possa enviar notificações de eventos de execução dos pipelines. Para isso, utilizaremos o serviço OCI Notifications!

Para aprofundar seu conhecimento neste serviço, acesse os links abaixo! 👇

- ❗ [Página oficial do OCI Notifications](https://www.oracle.com/devops/notifications/)
- 🧾 [Documentação do OCI Notifications](https://docs.oracle.com/en-us/iaas/Content/Notification/home.htm)

1. Na OCI, no menu de hambúrguer 🍔, acesse: **Developer Services** → **Application Integration** → **Notifications**.

![](./Images/notifications_create1.png)

2. Então, clique em **Create Topic**.

![](./Images/notifications_create2.png)

3. Atribua um nome ao tópico e clique em **Create**.

![](./Images/notifications_create3.png)

### <a name="Passo1.4"></a> Passo 1.4: Criação de External Connection
Nessa etapa, vamos configurar propriamente a conexão de um projeto do OCI DevOps ao repositório no GitHub.

1. Na OCI, no menu de hambúrguer 🍔, acesse: **Developer Services** → **DevOps** → **Projects**.
  
![](./Images/014-LAB4.png)

2. No compartment correspondente, clique em **Create DevOps Project**.

![](./Images/create_project.png)

3. Atribua um nome ao projeto, selecione o Notification Topic criado anteriormente e clique em **Create DevOps Project**.

![](./Images/devops_create1.png)

4. Clique no projeto DevOps criado e, na página do projeto, clique em **External Connections**.

![](./Images/external_connection1.png)

5. Clique em **Create external connection**.

![](./Images/external_connection2.png)

6. Atribua um nome à conexão, selecione o Vault e o Secret criados anteriormente, e clique em **Create**.

![](./Images/external_connection3.png)

### <a name="Passo1.5"></a> Passo 1.5: Repo Github espelhado

1. Na página do projeto, clique em **Code Repositories**.

![](./Images/code_repositories.png)

4. Clique em **Mirror repository**.

![](./Images/mirror_repo1.png)

5. Selecione, em 'Connection', a conexão criada anteriormente, em 'Repository', o repo no GitHub, e então clique em **Mirror Repository**.

![](./Images/mirror_repo2.png)

![](./Images/mirrored_repo.png)

Com isso, concluímos o espelhamento do repo no GitHub para o projeto OCI DevOps! Podemos seguir agora com os próximos passos!

- - -

## <a name="Passo2"></a> Passo 2: Criar pipeline de build e configurar criação da container image (CI)

1. Retorne à página inicial do projeto OCI DevOps.

2. Clique em **Create build pipeline**. 

 ![](./Images/build_pipe1.png)

3. Preencha o formulário da seguinte forma, e clique em **Create**:
   - **Name**: build
   - **Description**: (Defina uma descrição qualquer).

![](./Images/021-LAB4.png)

4. Acesse a aba de **Build Pipeline**, e clique em **Add Stage**.

![](./Images/023-LAB4.png)

5. Selecione a opção **Managed Build** e clique **Next**.

![](./Images/024-LAB4.png)

6. Preencha o formulário da seguinte forma:

- **Stage Name**: Criacao de container image
- **Description**: (Defina uma descrição qualquer).
- **OCI build agent compute shape**: *Não alterar*.
- **Base container image**: *Não alterar*.
- **Build spec file path**: *Não alterar*.
      
![](./Images/build_pipe3.png)

7. Em Primary code repository, clique em **Select**.

![](./Images/primary_code1.png)

8. Selecione as opções abaixo e clique em **Save**.

- **Source Connection type**: GitHub
- **Name**: *Selecione a conexão criada*.
- **Select Repository**: *Selecione o repo do github*.
- **Branch**: main.
- **Build source name**: storefront

![](./Images/primary_code2.png)

9. Feito isto, clique em **Add**.

![](./Images/primary_code3.png)

🤔 Neste momento é importante entender a forma como a ferramenta trabalha 📝.
    
- A ferramenta utiliza um documento no formato YAML para definir os passos que devem ser executados durante o processo de construção da aplicação.
- Por padrão este documento é chamado de *build_spec.yaml* e deve ser configurado previamente de acordo com as necessidades da aplicação.
- Os passos serão então executados por uma instância temporária (agent), que será provisionada no início de cada execução e destruída ao final do processo.
- 🧾 [Documentação de como formatar o documento de build](https://docs.oracle.com/pt-br/iaas/Content/devops/using/build_specs.htm)
- 📑 [Documento utilizado neste workshop (build_spec.yaml)](https://raw.githubusercontent.com/CeInnovationTeam/BackendFTDev/main/build_spec.yaml)

- - -

## <a name="Passo3"></a> Passo 3: Configurar entrega da container image (CI)

1. Na aba de Build Pipeline, clique no sinal de **"+"**, abaixo do stage **Criacao de container image**, e em **Add Stage**.
     
![](./Images/deliver_image1.png)


2. Selecione a opção **Deliver Artifacts** e clique em **Next**.
     
![](./Images/028-LAB4.png)

3. Preencha o formulário como abaixo e clique em **Create Artifact**.
 - **Stage name**: Entrega de Container Image
 - **Description**: (Defina uma descrição qualquer).

![](./Images/deliver_image2.png)
 
4. Preencha o formulário como abaixo e clique em **Add**.
- **Name**: mushop-demo
- **Type**: Container image repository
- **Artifact Source**: `<código-de-região>.ocir.io/<tenancy-namespace>/mushop-demo:${BUILDRUN_HASH}`
- **Replace parameters used in this artifact**: Yes, substitute placeholders
       
![](./Images/deliver_image3.png)

5. Finalmente, preencha como abaixo e clique em **Add**.

 - **Build config/result artifact name**: mushop-demo
       
![](./Images/deliver_image4.png)

![](./Images/build_part2.png)

<a name="FinalPasso3"></a> Isso conclui a parte de Build (CI) do projeto! Até aqui automatizamos a compilação do código, configuramos a criação da container image e também o seu armazenamento no container registry! Vamos agora para a parte de Deployment (CD)!

- - -

## <a name="Passo4"></a> Passo 4: Criar pipeline de deploy e configurar entrega da aplicação ao cluster Kubernetes (CD)

### <a name="Passo4.1"></a> Passo 4.1: Criação do kubernetes secret

Nesse momento, iremos inicialmente criar o [kubernetes secret](https://kubernetes.io/docs/concepts/configuration/secret/), para permitir que o cluster colete a container image durante o deployment. Iremos criar o secret no namespace 'mushop'.

1. Copie o comando abaixo para o seu bloco de notas e o edite substituindo os campos destacados por '<>'.

 ```shell
kubectl create secret docker-registry ocisecret --docker-server=iad.ocir.io --docker-username='<tenancy-namespace>/oracleidentitycloudservice/<e-mail>' --docker-password='<auth-token>' --docker-email='<e-mail>' -n mushop
 ```

2. Então, no **Cloud Shell**, execute o comando anterior.

![](./Images/secret_created.png)

### <a name="Passo4.2"></a> Passo 4.2: Adição de Environment no OCI DevOps
Vamos agora adicionar o cluster kubernetes como ambiente alvo no projeto OCI DevOps.

1. Retorne ao seu projeto DevOps clicando no menu hambúrguer 🍔 e acessando: **Developer Services**  → **Projects**.

1. No canto esquerdo, selecione **Environments**.
         
![](./Images/040-LAB4.png)

3. Clique em **Create New Environment**.

4. Preencha o formulário como abaixo e clique em **Next**.
  - **Environment type**: Oracle Kubernetes Engine
  - **Name**: OKE
  - **Description**: OKE

5. Selecione o cluster kubernetes, e clique em **Create Envrinoment**.

![](./Images/041-LAB4.png)

### <a name="Passo4.3"></a> Passo 4.3: Criação de artefato de deployment

Nesse momento, vamos adicionar o arquivo deployment.yaml no artifact registry, para utilizá-lo no pipeline de deployment.

Para aprofundar seu conhecimento neste serviço, acesse o link abaixo! 👇

- 🧾 [Documentação do OCI Artifact Registry](https://docs.oracle.com/en-us/iaas/Content/artifacts/home.htm)

1. Copie o [deployment.yaml](./scripts/deployment.yaml) para um novo bloco de notas e salve o arquivo.

2. Na OCI, no menu de hambúrguer 🍔, acesse: **Developer Services** → **Containers & Artifacts** → **Artifact Registry**.

![](./Images/artifact_registry1.png)

3. Clique em **Create Repository**.

![](./Images/artifact_registry2.png)

6. No canto esquerdo selecione **Artifacts** em seguida em **Add Artifact**.
          
![](./Images/042-LAB4.png)

7. Preencha o formulario como abaixo e clique em **Add**.
 - **Name**: deployment.yaml
 - **Type**: Kubernetes manifest
 - **Artifact Source**: Inline
 - **Value**: Cole o conteúdo do arquivo https://github.com/CeInnovationTeam/BackendFTDev/blob/main/scripts/deployment.yaml
 *Não altere a identação (espaços) do documento, pois isso pode quebrá-lo*.
 - **Replace parameters used in this artifact**: Yes, substitute placeholders
          
![](./Images/043_0-LAB4.png)

8. No canto esquerdo, selecione **Deployment Pipelines** e, em seguida, clique em **Create Pipeline**.
          
![](./Images/044-LAB4.png)

9. Preencha o formulário como abaixo e clique em **Create pipeline**.
 - **Pipeline name**: deploy
 - **Description**: (Defina uma descrição qualquer).
          
![](./Images/048-LAB4.png) 
          
![](./Images/049-LAB4.png)

10. Retorne à aba de **Pipeline** e clique em **Add Stage**.
          
![](./Images/050-LAB4.png)

11. Selecione a Opção **Apply Manifest to your Kubernetes Cluster** e clique em **Next**.
          
![](./Images/051-LAB4.png)

12. Preencha o formulário da seguinte forma:
 - **Name**: Deployment da Aplicacao
 - **Description**: (Defina uma Descrição qualquer).
 - **Environment**: OKE

![](./Images/052_0-LAB4.png)

13. Clique em **Select Artifact**, e selecione **deployment.yaml**.

![](./Images/052_1-LAB4.png)

14. Feito isto, clique em **Add**.
 
Com isso finalizamos a parte de Deployment (CD) do nosso projeto! No passo a seguir vamos conectar ambos os pipelines, e definir um gatilho (trigger) para que o processo automatizado se inicie!

- - -

## <a name="Passo5"></a> Passo 5: Criação do trigger de início e do trigger de conexão dos pipelines de CI e de CD

1. Retorne ao projeto clicando no 🍔 menu hambúrguer e acessando: **Developer Services**  → **Projects**.
  2. No canto esquerdo selecione **Triggers**, e em seguida clique em **Create Trigger**.

  ![](./Images/053-LAB4.png)

  3. Preencha o formulário como abaixo e clique em **Create**.
  - **Name**: Inicio
  - **Description**: (Defina uma descrição qualquer).
  - **Source connection**: OCI Code Repository
  - **Select code repository**: ftRepo
  - **Actions**: Add Action
    - **Select Build Pipeline**: build
    - **Event**: Push (check) 
    - **Source branch**: main

![](./Images/054-LAB4.png)

*A partir desse momento, qualquer novo push feito no repositório do projeto iniciará o pipeline de build criado nesse workshop*.

4. Retorne à configuração do pipeline de build do projeto selecionando **Build Pipelines** → **build**.

![](./Images/055-LAB4.png)

  5. Na aba de Build Pipeline, clique no sinal de **"+"** abaixo do stage **Entrega de Imagem de Container** e clique em **Add Stage**.

![](./Images/056-LAB4.png)

6. Selecione o item de **Trigger Deployment**, e clique em **Next**.

![](./Images/057-LAB4.png)

7. Preencha o formulário como abaixo e clique em **Add**.
- **Nome**: Inicio de Deployment
- **Description**: (Defina uma descrição qualquer).
- **Select deployment pipeline**: deploy

*Mantenha os demais campos sem alteração*.

![](./Images/058-LAB4.png)

Parabéns por chegar até aqui!! Nosso pipeline já está pronto! No próximo passo iremos validar o projeto, checando se está tudo ok.

 ## <a name="Passo6"></a> Passo 6: Execução e testes
  1.  Retorne ao projeto clicando no 🍔 menu hambúrguer e acessando: **Developer Services**  → **Projects**.
  2.  Retorne à configuração do pipeline de build do projeto selecionando **Build Pipelines** → **build**.
  
  ![](./Images/055-LAB4.png)

  3. No canto direito superior, selecione **Start Manual Run**.

![](./Images/055_1-LAB4.png)

  4. Mantenha as informações do formulário padrão, e clique em **Start Manual Run**.
  5. Aguarde a execução do fluxo.
  6. Acesse novamente o Cloud Shell e execute o comando abaixo.

  ```shell
  kubectl get svc
  ```

  7. Copie a informação de EXTERNAL-IP do serviço _svc-java-app_ assim que estiver disponível.

```shell
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
kubernetes     ClusterIP      10.96.0.1       <none>            443/TCP          30h
svc-app        LoadBalancer   10.96.252.115   <svc-app-ip>   80:31159/TCP     29h
svc-java-app   LoadBalancer   10.96.16.229    <EXTERNAL-IP>   8081:32344/TCP   103m
```

8. No **Cloud Shell**, execute o comando abaixo substituindo a informação de `<EXTERNAL-IP>` pelo IP copiado.

### 👏🏻 Parabéns!!! Você foi capaz de construir com sucesso um pipeline completo de **DevOps** na OCI para a aplicação MuShop! 🚀