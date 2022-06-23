# OCI Speech

O OCI Speech permite que os desenvolvedores extraiam texto de áudio usando modelos ASR (Automatic Speech Recognition) pré-treinados prontos para produção. O serviço de fala da OCI fornece transcrição automatizada e precisa em escala, sem exigir nenhum conhecimento em aprendizado de máquina. Ele pode ser acessado por meio de APIs REST e SDKs.

## Crie um bucket e faça upload de arquivos de áudio

Para usar o serviço de fala OCI (Oracle Cloud Infrastructure), você deve fazer upload de arquivos de áudio formatados para um bucket de armazenamento de objetos OCI.

### Formate arquivos de áudio no formato adequado

A OCI Speech suporta arquivos de áudio PCM WAV de 16 bits de canal único com uma taxa de amostragem de 16kHz. Recomendamos Audacity (GUI) ou ffmpeg (linha de comando) para transcodificação de áudio. Se você tiver arquivos de áudio que não estão na codificação compatível, instale o ffmpeg e execute o seguinte comando: 

```
ffmpeg -y -i <path to input file> -map 0:a -ac 1 -ar 16000 -b:a 16000 -acodec pcm_s16le <caminho para saída do arquivo wav>
```

Em alguns casos raros, um arquivo WAV pode ter a codificação correta, mas ter um cabeçalho de metadados diferente. Para corrigir isso:

```
ffmpeg -i <path to input file> -c copy -fflags +bitexact -flags:v +bitexact -flags:a +bitexact <path to output wav file>
```

Se seus arquivos de áudio não estiverem no formato WAV: os usuários da GUI podem usar qualquer software de edição de áudio que possa carregar seu arquivo de entrada e salvá-lo no formato .WAV. Para cenários automatizados ou de linha de comando, recomendamos usar o utilitário ffmpeg com o seguinte comando:

```
ffmpeg -i <input.ext> -fflags +bitexact -acodec pcm_s16le -ac 1 -ar 16000 <output.wav>
```
Se precisar de ajuda para instalar o ffmepg você pode seguir [este tutorial](https://www.geeksforgeeks.org/how-to-install-ffmpeg-on-windows/)

Como alternativa, baixe o arquivo de áudio de amostra pré-formatado para utilizar neste laboratório:

[exemplo áudio](https://objectstorage.sa-saopaulo-1.oraclecloud.com/n/grhuxslsficl/b/speech/o/nutricao_pets_reporter_pet.wav)

### Carregar arquivos para o object storage

Você precisa carregar os arquivos de áudio no Object storage Oracle, para serem usados ​​nos trabalhos de transcrição nas próximas etapas.

1 - Criar um bucket do Object Storage (esta etapa é opcional caso o bucket já tenha sido criado)

Primeiro, no menu OCI Services, clique em Object Storage.

![](./Images/IMG_001.PNG)

Em seguida, selecione Compartimento no menu suspenso à esquerda. Escolha o compartimento que corresponde ao seu nome ou nome da empresa.

![](./Images/IMG_002.PNG)

Em seguida, clique em Criar bucket.

![](./Images/IMG_003.PNG)

Em seguida, preencha a caixa de diálogo:

Nome do bucket: forneça um nome
Camada de armazenamento: STANDARD

Em seguida, clique em Criar

![](./Images/IMG_004.PNG)

2 - Carregar arquivo de áudio no bucket de armazenamento

clique no nome do bucket.

Selecione ou arraste o arquivo e clique em carregar

![](./Images/IMG_005.PNG)

