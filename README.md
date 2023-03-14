# OpenLDAP CSIC - Helm Chart

Baseado no helm chart do [fluktuid](https://artifacthub.io/packages/helm/fluktuid/openldap), o qual utiliza a imagem [osixia/openldap](https://github.com/osixia/docker-openldap) e no helm chart do [gabibbo97](https://artifacthub.io/packages/helm/gabibbo97/ldap-account-manager), o qual utiliza a imagem [ldap account manager](https://hub.docker.com/r/ldapaccountmanager/lam).

Portanto, para informações/configurações adicionais, como variáveis de ambiente, pode-se acessar esses links.

---

## Arquivo `values.yaml`

Para utilizar uma versão mais personalizada do openldap, é preciso haver um arquivo `values.yaml` no repositório do gitlab e modificar as seguintes configurações.

### Variáveis de Ambiente + senhas:

Primeiramente, deve-se alterar os valores das variáveis de ambiente de acordo com seu contexto. Por exemplo:

```
env:
  LDAP_LOG_LEVEL: "256"
  LDAP_ORGANIZATION: "CTIC"
  LDAP_DOMAIN: "ctic"
  LDAP_BASE_DN: "dc=ctic"
  LDAP_READONLY_USER: "false"
  LDAP_RFC2307BIS_SCHEMA: "false"
  LDAP_BACKEND: "mdb"
  KEEP_EXISTING_CONFIG: "false"
  LDAP_REMOVE_CONFIG_AFTER_SETUP: "true"
  LDAP_SSL_HELPER_PREFIX: "ldap"

adminPassword: 123
configPassword: 12345
```

OBS: Além das senhas, é preciso estar atento aos valores definidos em LDAP_ORGANIZATION, LDAP_DOMAIN e LDAP_BASE_DN, pois serão utilizados para criar o usuário admin do ldap.

---

### Persistência de Dados

A persistência de dados está configurada para `/var/lib/ldap` e `/etc/ldap/slapd.d`, mas é preciso que os campos em `persistence` estejam definidos. Exemplo:

```
persistence:
  enabled: true
  storage: 300Mi
```

Ou seja, é preciso habilitar a persistência e definir um tamanho. Porém, caso o `storage` não seja definido, será utilizado o padrão de 100Mi.

Além disso, caso já existam storage classes e PVCs no cluster, pode-se utilizá-los ao adicionar os seguintes campos:

```
persistence:
  enabled: true
  storage: 300Mi
  existingClaim: "claimName"
  storageClass: "storageClassName"
```

Em que `claimName` e `storageClassName` devem ser substituídos pelos nomes corretos do PVC e do SC, respectivamente.

---

### Ingress

Caso o ingress já esteja configurado no cluster, é possível utilizá-lo por meio das seguintes configurações:

```
ingress:
  enabled: true
  className: nginx-example
  host: ldap.com
  ldap:
    path: /
    pathType: Prefix
    serviceName: openldap
    servicePort: 389
  lam:
    path: /lam
    pathType: Prefix
    serviceName: openldap-lam
    servicePort: 80
```

Em que:
- `className` é o nome do ingress class. Indicar caso haja mais de um e/ou não tenha um definido por padrão.
- `host` é a url indicada para acessar os serviços
- `path` é a rota para um serviço específico
- `pathType: Prefix` indica que a rota deve corresponder ao prefixo indicado em path, ignora `/` após o prefixo, por exemplo. Ver mais na [documentação](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types).
- `serviceName` é o nome dado ao service utilizado pelo pod
- `servicePort` é a porta utilizada por aquele service

OBS: Apenas os campos `host` e `path` devem ser modificados.

OBS2: Para acessar os serviços, é preciso mapear esse `host` no arquivo `/etc/hosts` com o IP do LoadBalancer do ingress e o `host` definido. Exemplo:

```
111.111.11.11 ldap.com
```

---

## Arquivos Ldif

Caso haja arquivos ldif para popular o ldap, é preciso:

1. Adicionar/modificar a seguinte linha para indicar a presença de arquivos ldif:

```
customLdifFiles: true
```

2. Criar uma pasta na raiz do diretório nomeada `ldifs` com os arquivos ldifs para popular o ldap.

OBS: Quando é criado um arquivo de backup a partir do comando `slapcat`, o usuário admin é incluído no início do arquivo. Porém, como foi dito na seção `Variáveis de ambiente`, o usuário admin é criado ao iniciar o container, logo, é preciso excluí-lo do arquivo de backup para não ocorrer conflito na hora de adicionar esse arquivo no ldap.

### Adicionar arquivos ldif

Para facilitar a adição dos arquivos ldif, pode-se criar um script para ser executado por meio das seguintes configurações no arquivo `values.yaml`:

```
configldap: 
  enabled: true
  script:
    ldap.sh: |
      slapadd -l /container/service/slapd/assets/config/bootstrap/ldif/custom/backup.ldif
```

Ou seja, ao habilitar o configldap, o script `ldap.sh` será criado com o conteúdo indicado após a `|`.
Esse arquivo será colocado dentro do container em `/configldap`. Assim, quando o pod estiver rodando, não precisaria acessá-lo para essa configuração inicial, basta executar o script criado com, por exemplo:

```
kubectl exec -it `kubectl get pods -o=name | grep openldap-ldap` -- bash /configldap/ldap.sh
```

# Automatizando o Processo de Deploy
Para automatizar o processo de deploy em um cluster Kubernetes, é necessário configurar duas ferramentas disponíveis. A primeira é a integração direta entre o Gitlab e o cluster em que o serviço irá ficar alocado; e a segunda é a já conhecida pipeline do Gitlab CI/CD.

## Integrando um Cluster Kubernetes ao Gitlab
Para integrar o cluster que irá conter o serviço do LDAP/LAM ao gitlab, primeiro vá na página inicial deste repositório e clique em `Infrastructure`, e logo após em `kubernetes clusters`. Como na imagem a seguir:

![ldap_git](https://gl.idc.ufpa.br/csic/migration/ldap/-/raw/master/images/ldap_home.png)

A tela a seguir irá aparecer, neste clique no botão `Connect a Cluster`, como a seguir:

![ldap_git_1](https://gl.idc.ufpa.br/csic/migration/ldap/-/raw/master/images/ldap_home_1.png)

A janela abaixo irá aparecer. Nesta, indique um nome significativo para o seu agente e clique em `create agent: ""`:

![ldap_git_2](https://gl.idc.ufpa.br/csic/migration/ldap/-/raw/master/images/ldap_home_2.png)

Por fim, clique em `Register`:
![ldap_git_3](https://gl.idc.ufpa.br/csic/migration/ldap/-/raw/master/images/ldap_home_3.png)

Feito tudo isso, a tela final irá aparecer, nesta há um conjunto de comandos que devem ser executados internamente ao seu cluster para que seja feita devidamente a associação com o Gitlab.

![ldap_git_4](https://gl.idc.ufpa.br/csic/migration/ldap/-/raw/master/images/ldap_home_4.png)

## Pipeline

A pipeline presente no repositório realiza o deploy desse helm chart a partir do [artifacts.hub](https://artifacthub.io/packages/helm/openldap-lam/openldap) e utiliza a imagem [dtzar/helm-kubectl](https://hub.docker.com/r/dtzar/helm-kubectl) para utilizar os comandos `helm` e `kubectl`.

- As primeiras linhas servem apenas para utilizar o agente configurado ao integrar o cluster com o gitlab:

```
- kubectl config get-contexts
- kubectl config use-context csic/migration/ldap:csic-services
```

- As seguintes adicionam o repositório do artifacts.hub e baixam os arquivos:

```
- helm repo add openldap-lam https://danilonicioka.github.io/openldap-lam/
- helm pull openldap-lam/openldap --untar
```

- Após isso, a pasta ldifs no gitlab é copiada para dentro da pasta em que os arquivos foram baixados, a openldap:
```
- cp -rfp ldifs openldap
```

- Por fim, realiza-se o deploy do openldap a partir da pasta openldap e do arquivo values.yaml presente no gitlab:
```
- helm upgrade openldap openldap --install --values=values.yaml
```