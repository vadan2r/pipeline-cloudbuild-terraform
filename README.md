# Criando um Pipeline de CI/CD com Cloud Build e Terraform

Este documento detalha o processo passo a passo para criar um pipeline de Integração Contínua e Entrega Contínua (CI/CD) usando o Google Cloud Build e Terraform. Este pipeline automatizará a implantação de sua infraestrutura GCP com Terraform, garantindo consistência e eficiência.

## Pré-requisitos

*   Você precisa ter uma conta do Google Cloud Platform (GCP).
*   Você precisa ter o Cloud Build API habilitado em seu projeto GCP.
*   Você precisa ter um repositório de código (GitHub, Cloud Source Repositories, Bitbucket) que contenha seus arquivos Terraform.
*   Você precisa ter o Terraform instalado localmente para configuração inicial (apenas para o Passo 1).
*   Você precisa ter o Cloud Storage bucket para Terraform state (recomendado).

## Passo 1: Configurando o Backend do Terraform (Opcional, mas Recomendado)

Armazenar o estado do Terraform remotamente é uma prática recomendada para colaboração e consistência. Vamos configurar o backend do Terraform para usar um bucket do Cloud Storage.

1.  **Crie um Bucket do Cloud Storage:**

    ```bash
    gsutil mb -l REGIAO gs://NOME_DO_BUCKET
    ```

    Substitua `REGIAO` pela região desejada para o bucket (ex: `us-central1`) e `NOME_DO_BUCKET` por um nome exclusivo para o seu bucket.

2.  **Configure o Backend no seu arquivo `terraform.tf`:**

    ```terraform
    terraform {
      required_version = ">= 1.0"  # ajuste para a versão do Terraform que você está usando
      backend "gcs" {
        bucket = "NOME_DO_BUCKET"
        prefix = "terraform/state" # Opcional: para organizar seus states
      }
    }
    ```

    Substitua `NOME_DO_BUCKET` pelo nome do bucket que você criou.

3.  **Inicialize o Terraform:**

    ```bash
    terraform init
    ```

    Isso irá baixar os plugins necessários e configurar o backend.  Responda `yes` quando solicitado para migrar o estado local para o backend remoto.

## Passo 2: Criando um Arquivo de Configuração do Cloud Build (`cloudbuild.yaml` ou `cloudbuild.json`)

Crie um arquivo `cloudbuild.yaml` (ou `cloudbuild.json`) na raiz do seu repositório Terraform. Este arquivo definirá os passos do seu pipeline de CI/CD.

```yaml
steps:
  # Validar a configuração Terraform
  - name: 'hashicorp/terraform:latest'
    entrypoint: 'terraform'
    args: ['validate']

  # Aplicar as mudanças Terraform
  - name: 'hashicorp/terraform:latest'
    entrypoint: 'terraform'
    args:
      - 'apply'
      - '-auto-approve' # Auto aprovar a aplicação
    env:
      - 'GOOGLE_APPLICATION_CREDENTIALS=/workspace/service-account.json' # caminho da service account criada no passo 3

options:
  substitution_option: 'ALLOW_LOOSE'  # Habilita substituições de variáveis

substitutions:
  _TF_VAR_PROJECT_ID: 'SEU_ID_DO_PROJETO' # Substitua pelo ID do seu projeto
```

**Explicação:**

*   **`steps`:** Define a sequência de passos que o Cloud Build irá executar.
*   **`name`:** Especifica a imagem do Docker a ser usada para executar o passo. Neste caso, estamos usando a imagem oficial do Terraform (`hashicorp/terraform:latest`).
*   **`entrypoint`:** Especifica o comando a ser executado dentro do container Docker. Neste caso, estamos usando `terraform`.
*   **`args`:** Define os argumentos a serem passados para o comando `terraform`.
    *   `validate`: Valida a sintaxe e a configuração do seu código Terraform.
    *   `apply -auto-approve`: Aplica as mudanças definidas no seu código Terraform sem solicitar confirmação. **Use com cautela em ambientes de produção!**
*   **`env`:** Define variáveis de ambiente necessárias para o Cloud Build. Neste caso, configuramos as credenciais da conta de serviço para permitir que o Cloud Build acesse seus recursos GCP.
*   **`substitutions`**: Permite definir variáveis que podem ser substituídas no arquivo `cloudbuild.yaml`. Isso é útil para parametrizar o pipeline. No caso, estamos definindo a variável `_TF_VAR_PROJECT_ID` que será usada para configurar o projeto no Terraform.
*    **`substitution_option`:**  Habilita substituições de variáveis nos arquivos de configuração.

**Considerações importantes:**

*   **`validate`:** Adicionar o passo de validação garante que apenas configurações válidas sejam aplicadas.
*   **`-auto-approve`:** Em ambientes de produção, evite o uso de `-auto-approve`. Use abordagens como planos do Terraform e aprovações manuais. Para ambientes de desenvolvimento, `-auto-approve` pode agilizar o processo.
*   **Versão Terraform:** Especifique a versão do Terraform utilizada no nome da imagem Docker para evitar surpresas com mudanças de versões (ex: `'hashicorp/terraform:1.1.0'`).

## Passo 3: Criando uma Conta de Serviço para o Cloud Build

Para que o Cloud Build possa interagir com seus recursos GCP, você precisa criar uma conta de serviço com as permissões apropriadas.

1.  **Acesse a página Contas de Serviço:** No console do GCP, vá para "IAM e Administrador" -> "Contas de serviço".
2.  **Crie uma Conta de Serviço:** Clique em "+ CRIAR CONTA DE SERVIÇO".
3.  **Detalhes da Conta de Serviço:**
    *   **Nome da conta de serviço:** Dê um nome descritivo à conta de serviço (ex: `cloud-build-terraform`).
    *   **ID da conta de serviço:** O ID será gerado automaticamente.
    *   **Descrição da conta de serviço:** (Opcional) Adicione uma descrição.
4.  **Conceda acesso ao projeto a esta conta de serviço:**
    *   **Papéis:** Conceda os papéis necessários para que o Cloud Build gerencie seus recursos. Os papéis comuns incluem:
        *   **Editor (roles/editor):** (Amplo, mas simplifica a configuração inicial)
        *   **Cloud Build Service Agent (roles/cloudbuild.builds.builder):** (Necessário para o Cloud Build)
        *   **Terraform Service Agent (roles/terraform.serviceAgent):** (Se usar o provedor oficial do Terraform)
        *   **Storage Admin (roles/storage.admin):** (Se usar o backend do Cloud Storage)
        *   **Outros papéis específicos aos recursos que você está provisionando:** (Ex: Compute Admin para instâncias Compute Engine, Cloud SQL Admin para bancos de dados Cloud SQL)
    *   Para uma abordagem mais granular, conceda apenas os papéis necessários para cada recurso que o Cloud Build precisa gerenciar.
5.  **Concluído:** Clique em "CONCLUÍDO" para criar a conta de serviço.
6.  **Crie uma chave JSON para a conta de serviço:**
    *   Na lista de contas de serviço, clique na conta de serviço que você criou.
    *   Vá para a aba "CHAVES".
    *   Clique em "ADICIONAR CHAVE" e selecione "Criar nova chave".
    *   Selecione "JSON" como o tipo de chave e clique em "CRIAR".
    *   Um arquivo JSON será baixado para sua máquina. **Mantenha este arquivo em segurança e não o compartilhe!**
7.  **Adicione o arquivo JSON ao seu repositório:**
    *   Crie um diretório chamado `/workspace/` na raiz do seu repositório Terraform.
    *   Adicione o arquivo JSON da sua conta de serviço ao diretório `/workspace/`. Renomeie o arquivo para `service-account.json`.
    *   **IMPORTANTE:** Adicione `/workspace/service-account.json` ao seu `.gitignore` para evitar que a chave seja acidentalmente versionada.

## Passo 4: Criando um Trigger do Cloud Build

Um trigger define quando o Cloud Build deve executar o pipeline. Vamos configurar um trigger para executar o pipeline sempre que houver um push para a branch `main` do seu repositório.

1.  **Acesse a página Triggers do Cloud Build:** No console do GCP, vá para "Cloud Build" -> "Triggers".
2.  **Crie um Trigger:** Clique em "+ CRIAR TRIGGER".
3.  **Configuração do Trigger:**
    *   **Nome:** Dê um nome descritivo ao trigger (ex: `terraform-ci-cd`).
    *   **Região:** Selecione a região onde o trigger será executado.
    *   **Evento:** Selecione "Push para uma branch".
    *   **Repositório:**
        *   Selecione o repositório que contém seu código Terraform. Se você não tiver conectado o repositório ao Cloud Build, você precisará autorizar o Cloud Build a acessar o repositório.
    *   **Branch:** Especifique a branch que acionará o pipeline (ex: `main`). Você também pode usar regex para definir padrões de branchs.
    *   **Configuração:** Selecione "Arquivo de configuração do Cloud Build (yaml ou json)".
    *   **Localização do arquivo de configuração do Cloud Build:** Deixe como `$repo_root/cloudbuild.yaml` (ou `$repo_root/cloudbuild.json` se você usou JSON).
4.  **Criar:** Clique em "CRIAR" para criar o trigger.

## Passo 5: Testando o Pipeline

1.  **Faça um push para a branch configurada:** Faça uma pequena alteração no seu código Terraform e faça um push para a branch `main` (ou a branch que você configurou no trigger).
2.  **Verifique a execução do Cloud Build:** Vá para a página "Histórico" do Cloud Build. Você deverá ver uma nova execução do build sendo iniciada.
3.  **Analise os logs:** Clique na execução do build para visualizar os logs e verificar se todos os passos foram executados com sucesso.
4.  **Verifique a infraestrutura:** Verifique se as alterações definidas no seu código Terraform foram aplicadas à sua infraestrutura GCP.

## Dicas e Boas Práticas

*   **Modularize seu código Terraform:** Divida seu código Terraform em módulos reutilizáveis para facilitar a manutenção e o gerenciamento.
*   **Use variáveis Terraform:** Use variáveis Terraform para parametrizar seu código e torná-lo mais flexível.
*   **Versionamento:** Utilize o versionamento de código para garantir a consistência e rastreabilidade das alterações.
*   **Planos do Terraform:** Em ambientes de produção, use `terraform plan` para revisar as mudanças antes de aplicá-las com `terraform apply`.  Implemente um processo de aprovação manual para os planos.
*   **Testes:** Incorpore testes automatizados ao seu pipeline para verificar a funcionalidade da sua infraestrutura.
*   **Monitoramento:** Monitore seu pipeline e sua infraestrutura para detectar e resolver problemas rapidamente.
*   **Autenticação:** Utilize contas de serviço com o mínimo de privilégios necessários para garantir a segurança.
*    **Terraform Cloud:** Considere usar o Terraform Cloud para gerenciamento de estado, colaboração e execução remota dos planos Terraform.

## Solução de Problemas

*   **Falha na validação do Terraform:**
    *   Verifique se a sintaxe do seu código Terraform está correta.
    *   Verifique se todas as variáveis necessárias estão definidas.
*   **Falha na aplicação do Terraform:**
    *   Verifique se a conta de serviço tem as permissões necessárias para criar e gerenciar os recursos.
    *   Verifique se há erros nos logs do Terraform.
    *   Verifique se há conflitos de estado.
*   **O trigger não está sendo acionado:**
    *   Verifique se o trigger está configurado corretamente.
    *   Verifique se o evento que aciona o trigger (ex: push para a branch `main`) ocorreu.
*    **O pipeline não encontra o arquivo de configuração cloudbuild.yaml:**
    *   Verifique se o arquivo está na raiz do repositório e com o nome correto (cloudbuild.yaml ou cloudbuild.json).

Seguindo este guia, você poderá criar um pipeline de CI/CD robusto com Cloud Build e Terraform para automatizar a implantação da sua infraestrutura GCP. Lembre-se de consultar a documentação oficial do Google Cloud e Terraform para obter informações mais detalhadas e atualizadas.
```

**Observações:**

*   **Segurança:** A segurança é fundamental. Revise cuidadosamente as permissões da conta de serviço.
*   **Automatização:** Este é um exemplo básico. A complexidade do seu pipeline dependerá das suas necessidades específicas.
*   **Testes:** Adicione testes para garantir a qualidade e estabilidade da sua infraestrutura.
*   **Revisões:** Revise periodicamente seu pipeline para garantir que ele esteja atualizado com as melhores práticas e suas necessidades de negócios.
*   **Terraform Cloud:** Para ambientes de produção, o Terraform Cloud oferece recursos avançados de gerenciamento de estado, colaboração e auditoria.

Este README fornece um guia completo para criar um pipeline CI/CD com Cloud Build e Terraform.
Adapte-o às suas necessidades específicas e consulte a documentação oficial do Google Cloud e Terraform para obter informações mais detalhadas.
