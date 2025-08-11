# Guia Prático: Servidor NFS e Provisionamento Dinâmico em Cluster Kind

### Projeto para configuração de um servidor **NFS** integrado a um cluster **Kubernetes** criado com Kind, utilizando o **NFS Subdir External Provisioner** para provisionamento dinâmico de volumes.  
### O processo pode ser executado em máquinas virtuais com distribuições baseadas em Red Hat.

---
# Requisitos mínimos
### • Um host com Docker instalado para executar o cluster Kind
### • Um host dedicado para o servidor NFS
---

# Instalação do Kubernetes e NFS Client

## Instalar binário da ferramenta *Kind*
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## Criar cluster com o arquivo kind-config.yaml
### O arquivo de configuração cria um mapeamento de porta extra no container do kind que será utilizada pelo serviço NodePort
```bash
kind create cluster --config kind-config.yaml --name lab-nfs

# Instalar kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Instalar Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

kubectl cluster-info --context kind-lab-nfs
```

## Instalar os utilitários do *NFS (Network File System)*
```bash
dnf install nfs-utils -y
```
---

# Configuração do servidor NFS
## Instalar os utilitários do NFS
```bash
dnf install nfs-utils -y
```
## Criar diretório que será exportado para armazenar os volumes e aplicar as permissões nobody:nobody para que qualquer subdiretório criado pelo provisionador tenha permissões compatíveis para leitura/escrita por pods que não rodam como root.
```bash
mkdir -p /data/export
chown nobody:nobody /data/export 
```

## Adicionar o diretório criado no arquivo */etc/exports*, indicando que ele será exportado para hosts remotos com as configurações:

`*` - qualquer host pode acessar remotamente  
`rw` - acesso de leitura e escrita  
`sync` - garante que o servidor não responderá às solicitações antes que as alterações feitas por requisições anteriores sejam gravadas em disco  
`no_subtree_check` - evita verificações em subdiretórios  
`no_root_squash` - permite que o usuário root no cliente mantenha os privilégios no servidor  

```bash
echo "/data/export *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -rav
```

## Habilitar os serviços essenciais para funcionamento do protocolo
`rpcbind` - redireciona a requisição do cliente para a porta correta para que possa se conectar no serviço solicitado  
`nfs-server` - principal serviço do nfs   
`nfs-mountd` - gerencia as solicitações de montagem sistemas compartilhados  
```bash
systemctl enable --now rpcbind nfs-server nfs-mountd
```
---
# Validação do volume
## Acessar o servidor kind e executar o comando com o ip do servidor NFS para checar se o diretório foi exportado corretamente:
```bash
showmount -e 0.0.0.0
```
### O comando deve retornar os diretórios exportados no servidor NFS

---
# Implementação do *nfs-subdir-external-provisioner*
```bash
# Adicionar repositório
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

# Alterar o parâmetro nfs.server com o ip do servidor NFS
helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  -n nfs \
  --create-namespace \
  --set nfs.server=0.0.0.0 \
  --set nfs.path=/data/export \
  --set nfs.storageClass.accessModes=ReadWriteMany
```
---

# Deploy do Nginx com volume NFS
```bash
kubectl apply -f pvc.yaml
```
## Um persistent volume foi criado dinamicamente no cluster e no servidor NFS foi gerado um subdiretório referenciando esse volume

## Acessar o servidor NFS e criar o arquivo index.html dentro do subdiretório gerado dinamicamente em /data/export
```bash
cat <<'EOF' > index.html
<!DOCTYPE html>
<html lang="pt-BR">
<body>
    <h1>Volume com NFS funcionando corretamente</h1>
</body>
</html>
EOF
```

## Finalizar a instalação do Nginx
### Deployment criado com a montagem do volume no diretório */usr/share/nginx/html*, dessa maneira o arquivo index.html original será substituído pela nova versão criada no servidor NFS  
### O serviço do tipo *NodePort* expõe a porta 30080 no node Kind, que recebe as solicitações mapeadas da porta 8080 do docker host, e redireciona essas solicitações para a porta 8080 do serviço interno no cluster, que por sua vez encaminha para a porta 80 do pod
```bash
kubectl apply -f deploy.yaml
kubectl apply -f service.yaml
```
---
# Testar aplicação
```bash
curl localhost:8080
```