# Processamento paralelo de Videos no BBB 2.2.0
Disponibilizo este codigo aos usuarios do serviço mConf.rnp.br que atualmente esta usando a versão 2.2.0 do BBB.
Crio esta documentação baseado no [MR](https://github.com/bigbluebutton/bigbluebutton/pull/8394), dos proprios devs do BBB.
Na versão 2.2.0 os videos são processados sequenciamente (um video por vez) pelo arquivo `rap-process-worker.rb`, executado pelo serviço `bbb-rap-process-worker1.service`.
O arquivo `rap-process-worker.multiprocess.rb` que disponibilizo aceita filtros para reduzir a quantidade de arquivos que ele vai processar (Ex. `-p "[7-8]$"`  que so processa arquivos cujo nome finalizam em 7 e 8).
Isto permite iniciar varias instancia (através de varios serviços) sem que elas "roubem" os arquivos das outras.

## Instalação
### Planejamento
Existem varias pastas `workerX` que contem os serviços que rodarão em paralelo.
O calculo que fizemos é que cada serviço ocupa em media 3 nucleos do CPU (Ex. 10 workers ocupariam 30 nucleos).
Escolha a quantidade de acordo com sua realidade.
### Copiando os Arquivos
```sh
# No servidor de Gravação, clone este repositorio e entre em sua pasta
git clone https://github.com/arthurd2/bbb-parallel-workers.git
cd bbb-parallel-workers

# ANTES DE QUALQUER COISA
# Teste para ver se a versão do seu `rap-process-worker.rb` esta bantendo com a nossa.
# O comando deve retornar vazio, caso contrario
diff rap-process-worker.rb-original /usr/local/bigbluebutton/core/scripts/rap-process-worker.rb

# Copie o arquivo de processamento
chmod 755 rap-process-worker.multiprocess.rb
cp rap-process-worker.multiprocess.rb /usr/local/bigbluebutton/core/scripts/

# Copie o arquivo de serviços da pasta desejada (de acordo com a quantidade de processos paralelos desejados):
# ALTERE O NOME DA PASTA WORKERSX
cp workersX/*.service /usr/lib/systemd/system/
```
### Parando o processo atual.
* Remover as dependencias
```sh
cp /usr/lib/systemd/system/bbb-record-core.target /usr/lib/systemd/system/.bbb-record-core.target.bkp
sed -i 's/bbb-rap-process-worker.service//g' /usr/lib/systemd/system/bbb-record-core.target
```
* Desativando o serviço
```sh
# Parando e desabilitando os serviços atuais
systemctl stop bbb-rap-process-worker.service
systemctl disable bbb-rap-process-worker.service
systemctl mask bbb-rap-process-worker.service
#Este ultimo stop é so pra garantir mesmo
systemctl stop bbb-rap-process-worker.service
```
### Configurando os novos serviços.
* Iniciando os serviços
```sh
# ALTERE O NOME DA PASTA WORKERSX
# Habilitando todos os servicoes
for i in `ls workersX/*.service`; do systemctl enable $i ; done
# Iniciando todos os serviços
for i in `ls workersX/*.service`; do systemctl start $i ; done
```
### Desistalando Tudo
* Voltando o Backup
```sh
cp /usr/lib/systemd/system/.bbb-record-core.target.bkp /usr/lib/systemd/system/bbb-record-core.target 
```
* Matando os servicos os serviços
```sh
# ALTERE O NOME DA PASTA WORKERSX
# Habilitando todos os servicoes
for i in `ls workersX/*.service`; do systemctl stop $i ; done
# Iniciando todos os serviços
for i in `ls workersX/*.service`; do systemctl mask $i ; done
# Desabilitando todos os serviços
for i in `ls workersX/*.service`; do systemctl disable $i ; done
```
* Voltando os serviços
```sh
systemctl enable bbb-rap-process-worker.service
systemctl start bbb-rap-process-worker.service
```
