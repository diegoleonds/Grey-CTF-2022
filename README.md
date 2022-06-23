# Grey CTF 2022

## Ferramentas utilizadas:

-  [Apktool](https://ibotpeaches.github.io/Apktool/install/)
Ferramenta para engenharia reversa de arquivos apk. Foi utilizada para descompilar a apk e recompilar novamente após as alterações feitas. Instalação:

```
sudo apt-get install apktool 
```

- [Android Studio](https://developer.android.com/studio?hl=pt&gclid=CjwKCAjw-8qVBhANEiwAfjXLrsKFsxQE9Puoyua35lKIzzop74UQr-O2Q_fcyBjb1qcg-8CQTPPNYRoCQTIQAvD_BwE&gclsrc=aw.ds)
IDE para desenvolvimento Android. Foi utilizada para editar os arquivos, ler o log e utilizar emulador Android com API 28. Instalação:

```
sudo snap install android-studio --classic
```

## Desafios
Foram resolvidos: [Memory Game 1](https://ctfd.nusgreyhats.org/challenges#%F0%9F%8D%AD%20Memory%20Game%20%28Part%201%29-38) e [Memory Game 2](https://ctfd.nusgreyhats.org/challenges#%F0%9F%A9%B8%20Memory%20Game%20%28Part%202%29-39), ambos compartilham o mesmo código fonte que pode ser baixado em qualquer um dos enunciados.

## Resolução

### Vale para ambos desafios

- Primeiro usei o comando **file** para verificar o tipo do arquivo

```
file memory-game.apk
```

- Após ver que se tratava de .zip (pode-se dizer que uma apk é um zip) tentei descompilar o arquivo com o **apktool**

```
apktool d memory-game.apk
```

- Esse comando gerou uma pasta **memory-game** com os arquivos da apk decompilada

### Memory Game 1

- Li a descrição do exercício e isso me chamou a atenção **Do you know where the image assets are stored? I've made a nice drawing for you**: por padrão em projetos Android imagens são guardadas nas pastas **drawable** e **drawable-sufixo** que se encontram dentro de **res**, então fui buscar nelas

```
cd memory-game/res/drawable-hdpi
ls
```

- No output apareceram vários arquivos de imagem mas o que me chamou a atenção foi o **flag.png**, abri o arquivo e lá estava a flag

### Memory Game 2
Para resolver tive que alterar arquivos **.smali** e para isso usei essa [tabela](http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html) como referência para entender melhor o código

- Ao ler o enunciado tive duas ideias alterar a lógica:
	- Das regras do jogo para não precisar jogar ou deixar o jogo fácil
	- Por trás do log, para o log me dar a flag logo no início

A princípio resolvi seguir pela primeira opção porque achei que seria mais fácil de verificar (dependendo da organização do projeto)

- Antes de analisar os .smali fui ver o **AndroidManifest.xml** (um arquivo de configuração de um app Android) mas não achei nada de interessante nele

- Fui ver os arquivos em **smali/com/snatik** onde está o código fonte do app:
	- **GameApplication** (outro arquivo de configuração padrão) não tinha nada que pudesse ajudar
	- Resolvi ignorar a **MainActivity** e todos os arquivos na pasta **fragment** pois são classes de view e normalmente não contém regras de negócio, fiz o mesmo com as pastas **ui** e **themes**
	- Decidi olhar os arquivos da pasta **model** que parecia ter as classes referentes as regras do jogo:
		- Ao ver o arquivo **BoardArrangment** o método **isPair** me chamou a atenção, mas ainda não era o suficiente para resolver o problema
		- O arquivo **BoardConfiguration** me chamou a atenção já que tinha atributos referentes a **dificuldade, tempo, e número de: linhas, colunas e cartas por linha**. Resolvi modificar ele.

- Segue a representação do construtor da classe do arquivo escolhido em **Java**:

``` java
public BoardConfiguration(int i) {
        this.difficulty = i;
        switch (i) {
            case 1:
                this.numTiles = 6;
                this.numTilesInRow = 3;
                this.numRows = 2;
                this.time = 60;
                return;
            case 2:
                this.numTiles = 12;
                this.numTilesInRow = 4;
                this.numRows = 3;
                this.time = 90;
                return;
            case 3:
                this.numTiles = 18;
                this.numTilesInRow = 6;
                this.numRows = 3;
                this.time = 120;
                return;
            case 4:
                this.numTiles = 28;
                this.numTilesInRow = 7;
                this.numRows = 4;
                this.time = 150;
                return;
            case 5:
                this.numTiles = 32;
                this.numTilesInRow = 8;
                this.numRows = 4;
                this.time = 180;
                return;
            case 6:
                this.numTiles = 50;
                this.numTilesInRow = 10;
                this.numRows = 5;
                this.time = 20;
                return;
            default:
                throw new IllegalArgumentException("Select one of predefined sizes");
        }
    }
```
Minha primeira ideia foi colocar os mesmos valores do case 1 no case 6 para fazer a maior dificuldade do jogo ter as mesmas características da menor. Em Java ficaria assim:

``` java
public BoardConfiguration(int i) {
        this.difficulty = i;
        switch (i) {
            case 1:
                this.numTiles = 6;
                this.numTilesInRow = 3;
                this.numRows = 2;
                this.time = 60;
                return;

            // resto do código

            case 6:
                this.numTiles = 6;
                this.numTilesInRow = 3;
                this.numRows = 2;
                this.time = 60;
                return;
            default:
                throw new IllegalArgumentException("Select one of predefined sizes");
        }
    }
```

que em **smali** ficou:

``` smali
.method public constructor <init>(I)V
    .locals 3

    .line 18
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

    .line 19
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->difficulty:I

    const/4 v0, 0x6

    const/4 v1, 0x4

    const/4 v2, 0x3

    packed-switch p1, :pswitch_data_0

    .line 58
    new-instance p1, Ljava/lang/IllegalArgumentException;

    const-string v0, "Select one of predefined sizes"

    invoke-direct {p1, v0}, Ljava/lang/IllegalArgumentException;-><init>(Ljava/lang/String;)V

    throw p1

    :pswitch_0
    const/16 p1, 0x32

    .line 52
    iput v0, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    .line 53
    iput v2, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    const/4 p1, 0x2

    .line 54
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0x3c

    .line 55
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    goto :goto_0

    :pswitch_1
    const/16 p1, 0x20

    .line 46
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    const/16 p1, 0x8

    .line 47
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    .line 48
    iput v1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0xb4

    .line 49
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    goto :goto_0

    :pswitch_2
    const/16 p1, 0x1c

    .line 40
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    const/4 p1, 0x7

    .line 41
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    .line 42
    iput v1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0x96

    .line 43
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    goto :goto_0

    :pswitch_3
    const/16 p1, 0x12

    .line 34
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    .line 35
    iput v0, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    .line 36
    iput v2, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0x78

    .line 37
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    goto :goto_0

    :pswitch_4
    const/16 p1, 0xc

    .line 28
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    .line 29
    iput v1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    .line 30
    iput v2, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0x5a

    .line 31
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    goto :goto_0

    .line 22
    :pswitch_5
    iput v0, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTiles:I

    .line 23
    iput v2, p0, Lcom/snatik/matches/model/BoardConfiguration;->numTilesInRow:I

    const/4 p1, 0x2

    .line 24
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->numRows:I

    const/16 p1, 0x3c

    .line 25
    iput p1, p0, Lcom/snatik/matches/model/BoardConfiguration;->time:I

    :goto_0
    return-void

    :pswitch_data_0
    .packed-switch 0x1
        :pswitch_5
        :pswitch_4
        :pswitch_3
        :pswitch_2
        :pswitch_1
        :pswitch_0
    .end packed-switch
.end method
```

- Após isso retornei ao **Manifest.xml** para adicionar uma configuração que me permitiria ver o log da apk na IDE: dentro da tag **application** coloquei **android:debuggable="true"**. Ficou da seguinte forma:

``` xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:compileSdkVersion="30" android:compileSdkVersionCodename="11" package="com.snatik.matches" platformBuildVersionCode="30" platformBuildVersionName="11">
    <application
        android:allowBackup="true"
        android:appComponentFactory="android.support.v4.app.CoreComponentFactory"
        android:icon="@drawable/icon" android:label="@string/app_name"
        android:name="com.snatik.matches.GameApplication"
        android:debuggable="true"
        android:theme="@style/AppTheme">

        <activity android:label="@string/app_name" android:name="com.snatik.matches.MainActivity" android:screenOrientation="landscape">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

- Gerei a apk com o comando:

```
apktool b memory-game -o game.apk
```
