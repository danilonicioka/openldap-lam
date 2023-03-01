# Helm Chart para Openldap

Baseado no helm chart do [fluktuid](https://artifacthub.io/packages/helm/fluktuid/openldap), o qual utiliza a imagem do [osixia](https://github.com/osixia/docker-openldap) e no helm chart do [gabibbo97](https://artifacthub.io/packages/helm/gabibbo97/ldap-account-manager), o qual utiliza a imagem oficial do ldap account manager

---

## Ingress

A implementação do [fluktuid](https://artifacthub.io/packages/helm/fluktuid/openldap) não utiliza o Ingress, então, foi necessário criar o arquivo `ingress.yaml` para configurá-lo. Para isso, foi utilizado como referência a documentação do [kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/#simple-fanout).

Desse modo, basta incluir no arquivo `values.yaml` os seguintes campos:

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
- `pathType: Prefix` indica que a rota deve corresponder ao prefixo indicado em path, ignora `/` após o prefixo, por exemplo. Ver mais na [documentação](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)
- `serviceName` é o nome dado ao service utilizado pelo pod
- `servicePort` é a porta utilizada por aquele service
