---
title: Resize de um Persistent Volume
tags:
  - Kubernets
enableToc: true
---
Você tem um PVC (PersistentVolumeClaim) de 8Gi no Kubernetes que está cheio e deseja expandi-lo para 16Gi sem tempo de inatividade e sem interromper o workload. 

Veja as possibilidades para fazer um resize do PV.
# Expansão automática

1. **Verificar se o PVC suporta a expansão**

	Em primeiro lugar, a StorageClass subjacente deve ter `allowVolumeExpansion: true`. 

	Pode verificar isso com:

	```shell
	kubectl get storageclass <your-storageclass-name> -o yaml
	```

	ou 

	```shell
	kubectl get storageclass -n all
	```

	Procure por:

	```yml
	allowVolumeExpansion: true
	```

	ou 

	```shell
	ALLOWVOLUMEEXPANSION
	true
	```

	Se não estiver definido, não é possível redimensionar dinamicamente sem recriar os objetos manualmente.

2. **Editar o PVC para solicitar mais tamanho**
   
Você pode corrigir ou editar diretamente o PVC:

- Opção 1:

	```shell
	kubectl edit pvc <seu-nome-do-pvc>
	```

	Isso abrirá o YAML em um editor.
	Encontre o campo `spec.resources.requests.storage` e altere-o:

	```yml
	spec:
	  resources:
	    requests:
	      storage: 8Gi # -> Change to 16Gi
	```

	Salvar e sair.

- Opção 2:

	```shell
	kubectl patch pvc <your-pvc-name> -p '{"spec": {"resources": {"requests": {"storage": "16Gi"}}}}'
	```


3. **Confirmar que o PVC foi atualizado**

	Verificar o estado:

	```shell
	kubectl get pvc <your-pvc-name>
	```

	É preciso ver:

	```shell
	NAME            STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
	your-pvc-name   Bound    your-pv  16Gi       RWO            your-sc-name   10d
	```

4. **Redimensionamento do Filesystem**

	**Importante**:

	A maioria dos volumes modernos do Kubernetes (como AWS EBS, GCP PD, Azure Disk e até mesmo drivers CSI) redimensionará automaticamente o sistema de arquivos quando o tamanho do volume aumentar.

	Caso contrário, o contêiner do seu aplicativo pode precisar de um redimensionamento do sistema de arquivos manualmente dentro do pod.

	Você pode verificar executando no pod:

	```shell
	kubectl exec -it <your-pod-name> -- /bin/sh
	```

	Depois, lá dentro:

	```shell
	df -h
	```

	Se continuar a mostrar 8Gi mesmo depois de redimensionar, talvez seja necessário executar (para ext4):

	```shell
	resize2fs /dev/<your-device>
	```

	> A maior parte dos controladores CSI tratam disso automaticamente, pelo que, normalmente, não é necessária qualquer ação.

	Em resumo:
	- A StorageClass deve suportar a expansão do volume.
	- Edite ou corrija o PVC para 16Gi.
	- O sistema de ficheiros pode expandir-se automaticamente (ou pode ser necessário um rápido resize2fs).
	- Não há tempo de inatividade se feito corretamente - apenas um redimensionamento suave.

---

# Expansão Manual

Você deseja redimensionar um PersistentVolume (PV) em um cluster AKS de 10Gi para um tamanho maior sem perder dados. O PVC (PersistentVolumeClaim) a esse PV, mas o PV não tem uma storageClass atribuída, o que impede o redimensionamento automático. Você também está usando uma versão do Helm para a carga de trabalho e deseja saber se é necessária uma nova implantação.

```shell
k get pv acqiocombr-wp-uploads-pv -n marketing
NAME                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
acqiocombr-wp-uploads-pv   10Gi       RWX            Retain           Bound    marketing/acqiocombr-wp-uploads-pvc                  <unset>                          467d
```

```shell
k get pvc acqiocombr-wp-uploads-pvc -n marketing     
NAME                        STATUS   VOLUME                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
acqiocombr-wp-uploads-pvc   Bound    acqiocombr-wp-uploads-pv   10Gi       RWX                           <unset>                 467d
```

## Análise

- **Sem StorageClass**: O PV e o PVC não têm nenhum storageClassName definido (unset), o que significa que o PV provavelmente foi criado manualmente ou sem provisionamento dinâmico. No Kubernetes, o redimensionamento automático requer uma storageClass com allowVolumeExpansion: true. Sem isso, você não pode simplesmente corrigir o PVC para solicitar um tamanho maior
- **Política de recuperação de retenção**: O PV tem uma política de recuperação de retenção, o que significa que ele não será excluído automaticamente quando o PVC for excluído, o que ajuda a proteger seus dados durante operações manuais.
- **Contexto do AKS**: No AKS, os PVs são normalmente apoiados por Discos do Azure ou Arquivos do Azure. Como o modo de acesso é RWX (ReadWriteMany), é provável que seja um compartilhamento de Arquivos do Azure, que dá suporte ao redimensionamento.
- **Preservação de dados**: Para evitar a perda de dados, é necessário fazer o backup dos dados antes de fazer alterações no PV ou no PVC, pois o redimensionamento envolve a criação de um novo PV com uma capacidade maior e a migração de dados.
- **Helm**: Talvez não seja necessária uma nova versão do Helm, a menos que o PVC ou PV esteja fortemente acoplado à configuração do gráfico do Helm. É provável que seja possível redimensionar o PV/PVC de forma independente e atualizar a versão do Helm, se necessário.
## Solução
Como o redimensionamento automático não é possível sem uma storageClass, será necessário redimensionar manualmente o PV criando um novo PV com uma capacidade maior, copiando os dados do PV existente e atualizando o PVC para vincular ao novo PV. Aqui está um guia passo a passo para redimensionar o PV sem perda de dados:

1. **Verificar o armazenamento de apoio**

	Confirme se o PV é suportado por Arquivos do Azure (provavelmente, dado o RWX).

	```shell
	kubectl describe pv acqiocombr-wp-uploads-pv -n marketing
	```

	Procure o campo Source na saída para confirmar que se trata de um recurso azureFile. Anote o **secretName**, **shareName** e **storageAccountName** para uso posterior.

	Exemplo:

	```shell
	kubectl describe pv acqiocombr-wp-uploads-pv -n marketing

	Name:            acqiocombr-wp-uploads-pv
	Labels:          app=acqiocombr-app
					app.kubernetes.io/managed-by=Helm
					version=d41a66f8ff3061c56af28a7490f8be7131e3df56
	Annotations:     meta.helm.sh/release-name: site-institucional
					meta.helm.sh/release-namespace: marketing
					pv.kubernetes.io/bound-by-controller: yes
	Finalizers:      [kubernetes.io/pv-protection]
	StorageClass:    
	Status:          Bound
	Claim:           marketing/acqiocombr-wp-uploads-pvc
	Reclaim Policy:  Retain
	Access Modes:    RWX
	VolumeMode:      Filesystem
	Capacity:        10Gi
	Node Affinity:   <none>
	Message:         
	Source:
		Type:             AzureFile (an Azure File Service mount on the host and bind mount to the pod)
		SecretName:       acqiocombr-storage-secret
		SecretNamespace:  
		ShareName:        new-wp-uploads
		ReadOnly:         false
	Events:               <none>
	```

2. **Cópia de segurança dos dados**

	Para garantir que não haja perda de dados, crie um backup dos dados no PV existente.
	
	Crie um pod temporário para montar o PVC existente e copiar os dados para um local seguro (por exemplo, um Armazenamento de Blobs do Azure ou outro PVC). Exemplo de pod YAML (backup-pod.yaml)

	```yml
	apiVersion: v1
	kind: Pod
	metadata:
		name: backup-pod
		namespace: marketing
	spec:
		containers:
		- name: backup
			image: ubuntu
			command: ["/bin/sh", "-c", "sleep 3600"]
			volumeMounts:
			- mountPath: "/source"
				name: source-volume
		volumes:
		- name: source-volume
			persistentVolumeClaim:
				claimName: acqiocombr-wp-uploads-pvc
	```

	Aplique o pod

	```shell
	kubectl apply -f backup-pod.yaml
	```

	Copie os dados para uma localização segura (por exemplo, utilizando a transferência de ficheiros de armazenamento az ou outra ferramenta). Exemplo:

	```shell
	kubectl exec -n marketing backup-pod -- bash -c "tar -cvf /tmp/backup.tar /source"
	kubectl cp marketing/backup-pod:/tmp/backup.tar ./backup.tar
	```

	Carregue o backup.tar para o Azure Blob Storage ou outra localização segura.

3. **Criar um novo PVC com maior capacidade**

	Como o PVC original não tem storageClass, crie um novo PVC com o tamanho desejado. Exemplo de PVC YAML (new-pvc.yaml)

	```yml
	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
		name: acqiocombr-wp-uploads-pvc-new
		namespace: marketing
	spec:
		accessModes:
		- ReadWriteMany
		resources:
			requests:
				storage: 20Gi  # Set your desired size here
		storageClassName: ""  # Explicitly set to empty for manual binding
	```

	Aplique o PVC

	```shell
	kubectl apply -f new-pvc.yaml
	```

4. **Criar um novo PV para o PVC maior**

	Crie um novo PV que corresponda aos requisitos do novo PVC e aponte para uma partilha de Ficheiros do Azure redimensionada. 

	Primeiro, redimensione o compartilhamento de Arquivos do Azure no portal do Azure ou usando o Azure CL

	```shell
	az storage share update --account-name <storageAccountName> --name <shareName> --quota 20  # Size in GiB
	```

	Crie um novo PV YAML (new-pv.yaml) com base na configuração do PV original:

	```yml
	apiVersion: v1
	kind: PersistentVolume
	metadata:
		name: acqiocombr-wp-uploads-pv-new
	spec:
		capacity:
			storage: 20Gi  # Match the new PVC size
		accessModes:
		- ReadWriteMany
		persistentVolumeReclaimPolicy: Retain
		azureFile:
			secretName: <secretName>  # From original PV
			shareName: <shareName>   # From original PV
			readOnly: false
	```

	Aplique o novo PV:

	```shell
	kubectl apply -f new-pv.yaml
	```

	O novo PVC deve se vincular automaticamente ao novo PV. Verifique:

	```shell
	kubectl get pvc acqiocombr-wp-uploads-pvc-new -n marketing
	```

5. **Restaurar dados no novo PV**

	Monte o novo PVC em um pod temporário para restaurar os dados.

	Exemplo de pod YAML (restore-pod.yaml)

	```yml
	apiVersion: v1
	kind: Pod
	metadata:
	name: restore-pod
	namespace: marketing
	spec:
	containers:
	- name: restore
		image: ubuntu
		command: ["/bin/sh", "-c", "sleep 3600"]
		volumeMounts:
		- mountPath: "/target"
		name: target-volume
	volumes:
	- name: target-volume
		persistentVolumeClaim:
		claimName: acqiocombr-wp-uploads-pvc-new
	```

	Aplique o Pod:

	```shell
	kubectl apply -f restore-pod.yaml
	```

	Copie os dados de backup para o novo PVC:

	```shell
	kubectl cp ./backup.tar marketing/restore-pod:/tmp/backup.tar
	kubectl exec -n marketing restore-pod -- bash -c "tar -xvf /tmp/backup.tar -C /target"
	```

	Verifique se os dados foram restaurados corretamente.

6. **Atualizar a workload para usar o novo PVC**

	Atualize a versão do Helm ou a configuração da carga de trabalho para usar o novo PVC (acqiocombr-wp-uploads-pvc-new).

	Se estiver usando o Helm, verifique o values.yaml ou o gráfico para obter a referência do PVC. Exemplo de atualização de valores do Helm:

	```yaml
	persistence:
	uploads:
		claimName: acqiocombr-wp-uploads-pvc-new
	```

	Atualize a versão do Helm:

	```shell
	helm upgrade <release-name> <chart-name> -n marketing -f values.yaml
	```

	Como alternativa, edite o deploy diretamente:

	```shell
	kubectl edit deployment <deployment-name> -n marketing
	```

# StorageClass no AKS

## Azure Files

Estes utilizam os Ficheiros do Azure como backend, permitindo o armazenamento de ficheiros partilhados (modo RWX). Bom para cargas de trabalho como ficheiros multimédia partilhados (por exemplo, uploads do WordPress).

| Name                  | Type   | Provisioner        | Tier     | Notes                                                                                           |
| --------------------- | ------ | ------------------ | -------- | ----------------------------------------------------------------------------------------------- |
| azurefile             | Legacy | file.csi.azure.com | Standard | Ficheiros Azure clássicos (podem utilizar configurações mais antigas do controlador na driver). |
| azurefile-csi         | CSI    | file.csi.azure.com | Standard | Versão moderna com suporte CSI. Prefiro esta versão ao azurefile.                               |
| azurefile-premium     | Legacy | file.csi.azure.com | Premium  | Ficheiro do Azure em armazenamento suportado por SSD (latência inferior). Legado.               |
| azurefile-csi-premium | CSI    | file.csi.azure.com | Premium  | Ficheiro Azure mais rápido e com suporte CSI. Ideal para RWX sensíveis ao desempenho.           |
- Use azurefile-csi ou azurefile-csi-premium daqui para frente - eles são baseados em CSI, que é o padrão do Kubernetes agora.
- Modo de acesso RWX: Vários pods em nós podem acessar o mesmo volume.


## Azure Disks

Estes utilizam os Discos do Azure como backend (armazenamento em bloco), geralmente para acesso de um único pod (modo RWO). Adequado para bases de dados, aplicações que necessitam de armazenamento em bloco persistente.

| Name                | Type   | Provisioner        | Tier     | Notes                                                                      |
| ------------------- | ------ | ------------------ | -------- | -------------------------------------------------------------------------- |
| default             | Legacy | disk.csi.azure.com | Standard | Classe de armazenamento predefinida. Provavelmente baseado em HDD padrão.  |
| managed             | Legacy | disk.csi.azure.com | Standard | Versão pré-CSI. Pode estar a utilizar a compatibilidade do plugin in-tree. |
| managed-csi         | CSI    | disk.csi.azure.com | Standard | Disco gerido pelo Azure com suporte de CSI. Utilize-o em vez de managed.   |
| managed-premium     | Legacy | disk.csi.azure.com | Premium  | Versão gerida com base em SSD. Legado.                                     |
| managed-csi-premium | CSI    | disk.csi.azure.com | Premium  | Disco gerido com base em SSD com suporte CSI. Melhor para o desempenho.    |
- Apenas modo de acesso RWO (o pod tem de estar no mesmo nó que o volume).
- Utilize variantes CSI (managed-csi, managed-csi-premium) quando possível.

## 🔍 Dicas
- Para armazenamento partilhado (RWX): Utilize o azurefile-csi-premium se o desempenho for importante; caso contrário, utilize o azurefile-csi.
- Para armazenamento em bloco (RWO): Utilize managed-csi-premium para SSD, managed-csi para standard.