# Helm Chart para Openldap

Baseado no helm chart do [fluktuid](https://artifacthub.io/packages/helm/fluktuid/openldap), o qual utiliza a imagem [osixia/openldap](https://github.com/osixia/docker-openldap) e no helm chart do [gabibbo97](https://artifacthub.io/packages/helm/gabibbo97/ldap-account-manager), o qual utiliza a imagem [ldap account manager](https://hub.docker.com/r/ldapaccountmanager/lam).

Portanto, para informações/configurações adicionais, como variáveis de ambiente, pode-se acessar esses links.

---

## Arquivo `values.yaml`

Para utilizar uma versão mais personalizada do openldap, é preciso haver um arquivo `values.yaml` no repositório do gitlab e modificar as seguintes configurações.

### Variáveis de Ambiente:

Primeiramente, deve-se alterar os valores das variáveis de ambiente de acordo com seu contexto. Por exemplo:

```
env:
  LDAP_LOG_LEVEL: "256"
  LDAP_ORGANISATION: "EXAMPLE"
  LDAP_DOMAIN: "example"
  LDAP_BASE_DN: "dc=example"
  LDAP_ADMIN_PASSWORD: "123"
  LDAP_CONFIG_PASSWORD: "12345"
  LDAP_READONLY_USER: "false"
  LDAP_RFC2307BIS_SCHEMA: "false"
  LDAP_BACKEND: "mdb"
  KEEP_EXISTING_CONFIG: "false"
  LDAP_REMOVE_CONFIG_AFTER_SETUP: "true"
  LDAP_SSL_HELPER_PREFIX: "ldap"
```

OBS: Além das senhas, é preciso estar atento aos valores definidos em LDAP_ORGANISATION, LDAP_DOMAIN e LDAP_BASE_DN, pois serão utilizados para criar o usuário admin do ldap.

---

### Persistência de Dados

A persistência de dados está configurada para `/var/lib/ldap` e `/etc/ldap/slapd.d`, mas é preciso que os campos em `persistence` estejam definidos. Exemplo:

```
persistence:
  enabled: true
  storage: 300Mi
```

Ou seja, é preciso habilitar a persistência e definir um tamanho. Porém, caso o `storage` não seja definido, será utilizado o padrão de 100Mi.

---

### Ingress

Caso o ingress já esteja configurado no cluster, é possível utilizá-lo por meio das seguintes configurações:

```
ingress:
  enabled: true
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
- `host` é a url indicada para acessar os serviços
- `path` é a rota para um serviço específico
- `pathType: Prefix` indica que a rota deve corresponder ao prefixo indicado em path, ignora `/` após o prefixo, por exemplo. Ver mais na [documentação](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types).
- `serviceName` é o nome dado ao service utilizado pelo pod
- `servicePort` é a porta utilizada por aquele service

OBS: Apenas os campos `host` e `path` devem ser modificados.

OBS2: Para acessar os serviços, é preciso mapear esse `host` no arquivo `/etc/hosts` com o IP do LoadBalancer do ingress e o `host` definido. Exemplo:

```
200.239.67.36 ldap.com
```

---

## Arquivos Ldif

Caso haja arquivos ldif para popular o ldap, é preciso:

1. Adicionar/modificar a seguinte linha para indicar a presença de arquivos ldif:

```
customLdifFiles: true
```

2. Criar uma pasta na raiz do diretório do helm chart nomeada `ldifs` com os arquivos ldifs para popular o ldap.

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
kubectl exec -it `k get pods -o=name | grep openldap-ldap` -- bash /configldap/ldap.sh
```