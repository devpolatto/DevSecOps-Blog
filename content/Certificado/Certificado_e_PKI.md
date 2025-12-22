---
title: Tudo o que você deve saber sobre certificados e PKI, mas tem medo de perguntar
tags:
  - Certificado
  - PKI
enableToc: true
---
Certificados e infraestrutura de chave pública (PKI) são difíceis. Não brinca, certo? Conheço muitas pessoas inteligentes que evitaram essa toca de coelho em particular. Pessoalmente, evitei-o durante muito tempo e senti alguma vergonha por não saber mais. O resultado óbvio foi um ciclo vicioso: eu tinha vergonha de fazer perguntas e nunca aprendi.

Eventualmente, fui forçado a aprender essas coisas por causa do que elas permitem: a PKI permite definir um sistema criptograficamente. É universal e neutro em termos de fornecedor. Ele funciona em qualquer lugar para que partes do seu sistema possam ser executadas em qualquer lugar e se comunicar com segurança. É conceitualmente simples e super flexível. Ele permite que você [use TLS](https://smallstep.com/blog/use-tls.html) e descarte VPNs. Você pode ignorar tudo sobre sua rede e ainda ter fortes características de segurança. É muito bom.

Agora que aprendi, lamento não ter feito isso antes. O PKI é realmente poderoso e muito interessante. A matemática é complicada e os padrões são estupidamente barrocos, mas os conceitos básicos são bastante simples. Os certificados são a melhor forma de identificar códigos e dispositivos, e a identidade é muito útil para segurança, monitoramento, métricas e um milhão de outras coisas. Usar certificados não é tão difícil. Não é mais difícil do que aprender um novo idioma ou banco de dados. É um pouco chato e mal documentado.

Este é o manual que faltava. Acho que a maioria dos engenheiros consegue entender todos os conceitos mais importantes e peculiaridades comuns em menos de uma hora. Esse é o nosso objetivo aqui. Uma hora é um investimento muito pequeno para aprender algo que você literalmente não consegue fazer de outra maneira.

Meus motivos são principalmente didáticos. Mas usarei dois projetos de código aberto que construímos em smallstep em várias demonstrações: o [step CLI](https://smallstep.com/cli) e [o step certificados](https://smallstep.com/certificates) . Para acompanhar, você pode `brew install step`obter os dois (veja [as instruções completas de instalação aqui](https://github.com/smallstep/cli#installing) ). Se você preferir o botão fácil, crie uma autoridade hospedada gratuitamente usando nossa oferta [do Certificate Manager .](https://smallstep.com/certificate-manager/)

Vamos começar com uma frase tl; dr: o objetivo dos certificados e da PKI é vincular nomes a chaves públicas. É isso. O resto são apenas detalhes de implementação.

 [Star smallstep/cli](https://github.com/smallstep/cli)[3,523](https://github.com/smallstep/cli/stargazers) [Star smallstep/certificates](https://github.com/smallstep/certificates)[6,244](https://github.com/smallstep/certificates/stargazers)

## Uma visão geral ampla e algumas palavras que você deve saber

Vou usar alguns termos técnicos, então vamos defini-los antes de começar.

Uma **entidade** é qualquer coisa que existe, mesmo que exista apenas lógica ou conceitualmente. Seu computador é uma entidade. O mesmo acontece com algum código que você escreveu. Então é você. O mesmo acontece com o burrito que você comeu no almoço. O mesmo acontece com o fantasma que você viu quando tinha seis anos - mesmo que sua mãe estivesse certa e fosse apenas uma invenção da sua imaginação.

Toda entidade tem uma **identidade** . Este é difícil de definir. Identidade é o que faz de você você, sabe? Nos computadores, a identidade é geralmente representada como um conjunto de atributos que descrevem alguma entidade: grupo, idade, localização, cor favorita, tamanho do sapato, o que quer que seja. Um **identificador** não é o mesmo que uma identidade. Em vez disso, é uma referência única a alguma entidade que possui uma identidade. Sou Mike, mas Mike não é minha identidade. É um **nome** – identificador e nome são sinônimos (pelo menos para nossos propósitos).

As entidades podem **alegar** que possuem algum nome específico. Outras entidades poderão autenticar essa afirmação, confirmando a sua veracidade. Mas uma **afirmação** não precisa estar relacionada a um nome: posso fazer uma afirmação sobre qualquer coisa: minha idade, sua idade, direitos de acesso, sentido da vida, etc. alegar.

Um **assinante** ou **entidade final** é uma entidade que participa de uma PKI e pode ser **objeto** de um certificado. Uma **autoridade de certificação** (CA) é uma entidade que emite certificados para assinantes – um **emissor** de certificado . Os certificados que pertencem aos assinantes são às vezes chamados de certificados de entidade final ou **certificados folha** por motivos que ficarão mais claros quando discutirmos as cadeias de certificados. Os certificados que pertencem a CAs são geralmente chamados de **certificados raiz** ou **certificados intermediários,** dependendo do tipo de CA. Finalmente, uma **parte confiável** é um usuário de certificado que verifica e confia nos certificados emitidos por uma CA. Para confundir um pouco as coisas, uma entidade pode ser tanto um assinante quanto uma parte confiável. Ou seja, uma única entidade pode ter o seu próprio certificado e utilizar outros certificados para autenticar pares remotos (é o que acontece com o TLS mútuo, por exemplo).

Isso é o suficiente para começarmos, mas se a pedagogia o entusiasma, considere colocar [a RFC 4949](https://tools.ietf.org/html/rfc4949) no seu Kindle. Para todos os outros, vamos ser concretos. Como fazemos reivindicações e autenticamos coisas na prática? Vamos falar de criptografia.

## MACs e assinaturas autenticam coisas[](https://smallstep.com/blog/everything-pki/#macs-and-signatures-authenticate-stuff)

Um **código de autenticação de mensagem** (MAC) é um dado usado para verificar qual entidade enviou uma mensagem e para garantir que uma mensagem não foi modificada. A ideia básica é alimentar um segredo compartilhado (uma senha) junto com uma mensagem por meio de uma função hash. A saída hash é um MAC. Você envia o MAC junto com a mensagem para algum destinatário.

Um destinatário que também conheça o segredo compartilhado pode produzir seu próprio MAC e compará-lo com o fornecido. As funções hash têm um contrato simples: se você alimentá-las com a mesma entrada duas vezes, obterá exatamente a mesma saída. Se a entrada for diferente – mesmo que por um único bit – a saída será totalmente diferente. Portanto, se o MAC do destinatário corresponder ao enviado com a mensagem, pode-se ter certeza de que a mensagem foi enviada por outra entidade que conhece o segredo compartilhado. Supondo que apenas entidades confiáveis ​​conheçam o segredo compartilhado, o destinatário pode confiar na mensagem.

As funções hash também são unilaterais: é computacionalmente inviável pegar a saída de uma função hash e reconstruir sua entrada. Isto é fundamental para manter a confidencialidade de um segredo compartilhado: caso contrário, algum intruso poderia bisbilhotar seus MACs, reverter sua função hash e descobrir seus segredos. Isso não é bom. A validade dessa propriedade depende criticamente de detalhes sutis de como as funções hash são usadas para construir MACs. Detalhes sutis que não vou entrar aqui. Portanto, cuidado: não tente inventar seu próprio algoritmo MAC. Utilize [HMAC](https://en.wikipedia.org/wiki/HMAC) .

![2018-12-11-hmac.jpg](https://smallstep.imgix.net/2018_12_11_hmac_f9ea28a836.jpg?auto=format%2Ccompress&fit=max&q=50)

Toda essa conversa sobre MACs é um prólogo: nossa verdadeira história começa com _assinaturas_ . Uma **assinatura** é conceitualmente semelhante a um MAC, mas em vez de usar um segredo compartilhado você usa um par de chaves (definido em breve). Com um MAC, pelo menos duas entidades precisam conhecer o segredo compartilhado: o remetente e o destinatário. Um MAC válido pode ter sido gerado por qualquer uma das partes e você não sabe qual. As assinaturas são diferentes. Uma assinatura pode ser verificada usando uma chave pública, mas só pode ser gerada com uma chave privada correspondente. Assim, um destinatário que possui apenas uma chave pública pode verificar assinaturas, mas não pode gerá-las. Isso lhe dá um controle mais rígido sobre quem pode assinar as coisas. Se apenas uma entidade conhece a chave privada, você obtém uma propriedade chamada **não repúdio** : o detentor da chave privada não pode negar (repudiar) o fato de ter assinado alguns dados.

Se você já está confuso, relaxe. Elas são chamadas de assinaturas por uma razão: são como assinaturas no mundo real. Você tem algumas coisas com as quais deseja que alguém concorde? Você quer ter certeza de que poderá provar que eles concordaram mais tarde? Legal. Escreva e peça que assinem.

## A criptografia de chave pública permite que os computadores vejam[](https://smallstep.com/blog/everything-pki/#public-key-cryptography-lets-computers-see)

Certificados e PKI são construídos em **criptografia de chave pública** (também chamada de **criptografia assimétrica** ), que utiliza **pares de chaves** . Um par de chaves consiste em uma **chave pública** que pode ser distribuída e compartilhada com o mundo, e uma **chave privada** correspondente que deve ser mantida confidencial pelo proprietário.

Vamos repetir a última parte porque é importante: a segurança de um criptossistema de chave pública depende de manter as chaves privadas privadas.

Há duas coisas que você pode fazer com um par de chaves:

- Você pode **criptografar** alguns dados com a chave pública. A única maneira de descriptografar esses dados é com a chave privada correspondente.
- Você pode **assinar** alguns dados com a chave privada. Qualquer pessoa que conheça a chave pública correspondente pode verificar a assinatura, comprovando qual a chave privada que a produziu.

A criptografia de chave pública é um presente mágico da matemática para a ciência da computação. [A matemática](https://www.math.auckland.ac.nz/~sgal018/crypto-book/crypto-book.html) é complicada, com certeza, mas você não precisa entendê-la para avaliar seu valor. A criptografia de chave pública permite que os computadores façam algo que de outra forma seria impossível: **a criptografia de chave pública permite que os computadores vejam** .

Ok, deixe-me explicar… a criptografia de chave pública permite que um computador (ou pedaço de código) prove a outro que sabe algo sem compartilhar esse conhecimento diretamente. Para provar que você conhece uma senha, você deve compartilhá-la. Quem quer que você compartilhe pode usá-lo sozinho. Não é assim com uma chave privada. É como a visão. Se você sabe como eu sou, você pode dizer quem eu sou – autenticar minha identidade – olhando para mim. Mas você não pode mudar de forma para se passar por mim.

A criptografia de chave pública faz algo semelhante. Se você conhece minha chave pública (como eu sou), você pode usá-la para me ver na rede. Você poderia me enviar um grande número aleatório, por exemplo. Posso assinar seu número e enviar minha assinatura. Verificar essa assinatura é uma boa prova de que você está falando comigo. Isso efetivamente permite que os computadores vejam com quem estão falando em uma rede. Isso é tão útil que consideramos isso um dado adquirido no mundo real. Através de uma rede é pura magia. Obrigado matemática.

## Certificados: carteiras de motorista para computadores e código[](https://smallstep.com/blog/everything-pki/#certificates-drivers-licenses-for-computers-and-code)

E se você ainda não souber minha chave pública? É para isso que servem os certificados.

Os certificados são fundamentalmente _muito_ simples. Um certificado é uma estrutura de dados que contém uma chave pública e um nome. A estrutura de dados é então _assinada_ . A assinatura _vincula_ a chave pública ao nome. A entidade que assina um certificado é chamada de **emissor** (ou autoridade de certificação) e a entidade nomeada no certificado é chamada de **sujeito** .

Se _Some Issuer_ assinar um certificado para _Bob_ , esse certificado pode ser interpretado como a declaração: " _Some Issuer_ diz que a chave pública de _Bob é 01:23:42...".Esta é uma afirmação feita por_ _Some Issuer_ sobre _Bob_ . A reivindicação é assinada por _Some Issuer_ , portanto, se você souber a chave pública de _Some Issuer,_ poderá autenticá-la verificando a assinatura. Se você confia _em algum emissor_ , pode confiar na reivindicação. Assim, os certificados permitem usar a confiança e o conhecimento da chave pública de um emissor para aprender a chave pública de outra entidade (neste caso, a de _Bob_ ). É isso. Fundamentalmente, isso é tudo que um certificado é.

![2018-12-11-drivers-license-cert.jpg](https://smallstep.imgix.net/2018_12_11_drivers_license_cert_10a66d465e.jpg?auto=format%2Ccompress&fit=max&q=50)

Os certificados são como carteiras de motorista ou passaportes para computadores e códigos. Se você nunca me conheceu antes, mas confia no Detran, pode usar minha licença para autenticação: verificar se a licença é válida (verificar holograma, etc), olhar a foto, olhar para mim, ler o nome. Os computadores usam certificados para fazer a mesma coisa: se você nunca conheceu algum computador antes, mas confia em alguma autoridade de certificação, você pode usar um certificado para autenticação: verificar se o certificado é válido (verificar assinatura, etc), olhar para público chave, "veja a chave privada" na rede (conforme descrito acima), leia o nome.

![2018-12-11-drivers-license-cert.jpg](https://smallstep.imgix.net/2018_12_11_drivers_license_cert_10a66d465e.jpg?auto=format%2Ccompress&fit=max&q=50)

Vamos dar uma olhada rápida em um certificado real:

![2018-12-11-licença-vs-cert.jpg](https://smallstep.imgix.net/2018_12_11_license_vs_cert_0b24213247.jpg?auto=format%2Ccompress&fit=max&q=50)

Sim, talvez eu tenha simplificado um pouco a história. Assim como a carteira de motorista, há outras coisas nos certificados. As licenças indicam se você é um doador de órgãos e se está autorizado a dirigir um veículo comercial. Os certificados informam se você é uma CA e se sua chave pública deve ser usada para assinatura ou criptografia. Ambos também têm vencimentos.

Há muitos detalhes aqui, mas isso não muda o que eu disse antes: fundamentalmente, um certificado é apenas algo que vincula uma chave pública a um nome.

## X.509, ASN.1, OIDs, DER, PEM, PKCS, meu Deus…[](https://smallstep.com/blog/everything-pki/#x509-asn1-oids-der-pem-pkcs-oh-my)

Vejamos como os certificados são representados como bits e bytes. Na verdade, esta parte é irritantemente complicada. Na verdade, suspeito que a maneira esotérica e mal definida como os certificados e as chaves são codificados seja a fonte da maior confusão e frustração em torno da PKI em geral. Essa coisa é idiota. Desculpe.

Normalmente, quando as pessoas falam sobre certificados sem qualificação adicional, estão se referindo aos certificados X.509 v3. Mais especificamente, eles geralmente estão falando sobre a variante PKIX descrita na [RFC 5280](https://tools.ietf.org/html/rfc5280) e refinada pelos [Requisitos de linha de base](https://cabforum.org/baseline-requirements-documents/) do CA/Browser Forum . Em outras palavras, eles estão se referindo ao tipo de certificado que os navegadores entendem e usam para HTTPS (HTTP sobre TLS). Existem outros formatos de certificado. Notavelmente, SSH e PGP têm os seus próprios. Mas vamos nos concentrar no X.509. Se você entender o X.509, será capaz de descobrir todo o resto.

Como esses certificados são amplamente suportados - eles têm boas bibliotecas e outros enfeites - eles também são frequentemente usados ​​em outros contextos. Eles são certamente o formato mais comum para certificados emitidos por PKI interna (definido em breve). É importante ressaltar que esses certificados funcionam imediatamente com clientes e servidores TLS e HTTPS.

Você não pode apreciar totalmente o X.509 sem uma pequena lição de história. O X.509 foi padronizado pela primeira vez em 1988 como parte do projeto mais amplo X.500 sob os auspícios da ITU-T (o órgão de padronização da União Internacional de Telecomunicações). O X.500 foi um esforço das empresas de telecomunicações para construir uma lista telefônica global. Isso nunca aconteceu, mas os vestígios permanecem. Se você já olhou para um certificado X.509 e se perguntou por que algo projetado para a web codifica uma localidade, um estado e um país, aqui está sua resposta: o X.509 não foi projetado para a web. Foi projetado há trinta anos para construir uma lista telefônica.

![2018-12-11-cert-nome-distinto.jpg](https://smallstep.imgix.net/2018_12_11_cert_distinguished_name_079422ef55.jpg?auto=format%2Ccompress&fit=max&q=50)

O X.509 baseia-se no ASN.1, outro padrão ITU-T (definido pelo X.208 e X.680). ASN significa Notação de Sintaxe Abstrata (1 significa Um). ASN.1 é uma notação para definir tipos de dados. Você pode pensar nisso como JSON para X.509, mas na verdade é mais como protobuf, parcimônia ou SQL DDL. RFC 5280 usa ASN.1 para definir um certificado X.509 como um objeto que contém vários bits de informação: um nome, chave, assinatura, etc.

ASN.1 possui tipos de dados normais como inteiros, strings, conjuntos e sequências. Ele também possui um tipo incomum que é importante entender: identificadores de objetos (OIDs). Um OID é como um URI, mas é mais irritante. Eles são (supostamente) identificadores universalmente exclusivos. Estruturalmente, os OIDs são uma sequência de números inteiros em um namespace hierárquico. Você pode usar um OID para _marcar_ um dado com um tipo. Uma string é apenas uma string, mas se eu marcar uma string com OID, `2.5.4.3`ela não será mais uma string comum - será um _nome comum_ X.509 .

![2018-12-11-oids.jpg](https://smallstep.imgix.net/2018_12_11_oids_ae59d1b6d5.jpg?auto=format%2Ccompress&fit=max&q=50)

ASN.1 é _abstrato_ no sentido de que o padrão não diz nada sobre como as coisas devem ser representadas como bits e bytes. Para isso existem várias _regras de codificação_ que especificam representações concretas para valores de dados ASN.1. É uma camada de abstração adicional que deveria ser útil, mas na maioria das vezes é apenas irritante. É como a diferença entre unicode e utf8 (eek).

Existem [várias regras de codificação](https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One#Encodings) para ASN.1, mas há apenas uma que é comumente usada para certificados X.509 e outras coisas criptográficas: regras de codificação distintas ou DER (embora as regras básicas de codificação (BER) não canônicas também sejam usadas ocasionalmente ). DER é uma codificação de tipo-comprimento-valor bastante simples, mas você realmente não precisa se preocupar com isso, pois as bibliotecas farão a maior parte do trabalho pesado.

Infelizmente, a história não para por aqui. Você não precisa se preocupar muito com a codificação e decodificação do DER, mas _definitivamente_ precisará descobrir se um certificado específico é um certificado X.509 codificado por DER simples ou algo mais sofisticado. Existem duas dimensões potenciais de fantasia: podemos estar a olhar para algo mais do que DER bruto, e podemos estar a olhar para algo mais do que apenas um certificado.

Começando com a dimensão anterior, DER é binário direto e é difícil copiar e colar dados binários e desviá-los pela web. Portanto, a maioria dos certificados é empacotada em arquivos [PEM](https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One#Encodings) (que significa _Privacy Enhanced EMail_ , outro estranho vestígio histórico). Se você já trabalhou com [MIME](https://en.wikipedia.org/wiki/MIME) , o PEM é semelhante: uma carga útil codificada em base64 imprensada entre um cabeçalho e um rodapé. O cabeçalho PEM possui um rótulo que supostamente descreve a carga útil. Surpreendentemente, esse trabalho simples é quase sempre mal feito e os rótulos PEM são frequentemente inconsistentes entre as ferramentas ( [a RFC 7468](https://tools.ietf.org/html/rfc7468) tenta padronizar o uso do PEM neste contexto, mas não é completa e nem sempre é seguida). Sem mais delongas, aqui está a aparência de um certificado X.509 v3 codificado por PEM:

`-----BEGIN CERTIFICATE----- MIIBwzCCAWqgAwIBAgIRAIi5QRl9kz1wb+SUP20gB1kwCgYIKoZIzj0EAwIwGzEZ MBcGA1UEAxMQTDVkIFRlc3QgUm9vdCBDQTAeFw0xODExMDYyMjA0MDNaFw0yODEx MDMyMjA0MDNaMCMxITAfBgNVBAMTGEw1ZCBUZXN0IEludGVybWVkaWF0ZSBDQTBZ MBMGByqGSM49AgEGCCqGSM49AwEHA0IABAST8h+JftPkPocZyuZ5CVuPUk3vUtgo cgRbkYk7Ong7ey/fM5fJdRNdeW6SouV5h3nF9JvYKEXuoymSNjGbKomjgYYwgYMw DgYDVR0PAQH/BAQDAgGmMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAS BgNVHRMBAf8ECDAGAQH/AgEAMB0GA1UdDgQWBBRc+LHppFk8sflIpm/XKpbNMwx3 SDAfBgNVHSMEGDAWgBTirEpzC7/gexnnz7ozjWKd71lz5DAKBggqhkjOPQQDAgNH ADBEAiAejDEfua7dud78lxWe9eYxYcM93mlUMFIzbWlOJzg+rgIgcdtU9wIKmn5q FU3iOiRP5VyLNmrsQD3/ItjUN1f1ouY= -----END CERTIFICATE-----`

Os certificados codificados por PEM geralmente carregam uma extensão `.pem`, `.crt`ou . `.cer`Um certificado bruto codificado usando DER geralmente carrega uma `.der`extensão. Novamente, não há muita consistência aqui, então sua milhagem pode variar.

Voltando à nossa outra dimensão de fantasia: além da codificação mais sofisticada usando PEM, um certificado pode ser embalado em embalagens mais sofisticadas. Vários _formatos de envelope_ definem estruturas de dados maiores (ainda usando ASN.1) que podem conter certificados, chaves e outras coisas. Algumas pessoas pedem “um certificado” quando na verdade querem um certificado em um desses envelopes. Então cuidado.

Os formatos de envelope que você provavelmente encontrará fazem parte de um conjunto de padrões chamado PKCS (Public Key Cryptography Standards) publicado pelos laboratórios RSA (na verdade, a história é [um pouco mais complicada](https://security.stackexchange.com/questions/73156/whats-the-difference-between-x-509-and-pkcs7-certificate) , mas tanto faz). O primeiro é [o PKCS#7](https://tools.ietf.org/html/rfc2315) , renomeado como [Cryptographic Message Syntax](https://tools.ietf.org/html/rfc5652) (CMS) pela IETF, que pode conter um ou mais certificados (codificando uma cadeia completa de certificados, descrita em breve). PKCS#7 é comumente usado por Java. Extensões comuns são `.p7b`e `.p7c`. O outro formato de envelope comum é [o PKCS#12,](https://tools.ietf.org/html/rfc7292) que pode conter uma cadeia de certificados (como PKCS#7) junto com uma chave privada (criptografada). PKCS#12 é comumente usado por produtos Microsoft. Extensões comuns são `.pfx`e `.p12`. Novamente, os envelopes PKCS#7 e PKCS#12 também usam ASN.1. Isso significa que ambos podem ser codificados como DER, BER ou PEM bruto. Dito isto, na minha experiência, eles são quase sempre DER brutos.

A codificação de chave é igualmente complicada, mas o padrão é geralmente o mesmo: alguma estrutura de dados ASN.1 descreve a chave, DER é usado como uma codificação binária e PEM (esperançosamente com um cabeçalho útil) pode ser usado como uma representação um pouco mais amigável. Decifrar o tipo de chave que você está vendo é metade arte, metade ciência. Se você tiver sorte, [o RFC 7468](https://tools.ietf.org/html/rfc7468) fornecerá uma boa orientação para descobrir qual é a sua carga útil do PEM. As chaves de curva elíptica são geralmente rotuladas como tal, embora [não pareça haver nenhuma padronização](https://tools.ietf.org/html/rfc5915#section-4) . Outras chaves são simplesmente "PRIVATE KEY" da PEM. Isso geralmente indica uma carga útil [PKCS#8](https://tools.ietf.org/html/rfc5208) , um envelope para chaves privadas que inclui o tipo de chave e outros metadados. Aqui está um exemplo de chave de curva elíptica codificada em PEM:

`$ step crypto keypair --kty EC --no-password --insecure ec.pub ec.prv $ cat ec.pub ec.prv -----BEGIN PUBLIC KEY----- MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEc73/+JOESKlqWlhf0UzcRjEe7inF uu2z1DWxr+2YRLfTaJOm9huerJCh71z5lugg+QVLZBedKGEff5jgTssXHg== -----END PUBLIC KEY----- -----BEGIN EC PRIVATE KEY----- MHcCAQEEICjpa3i7ICHSIqZPZfkJpcRim/EAmUtMFGJg6QjkMqDMoAoGCCqGSM49 AwEHoUQDQgAEc73/+JOESKlqWlhf0UzcRjEe7inFuu2z1DWxr+2YRLfTaJOm9hue rJCh71z5lugg+QVLZBedKGEff5jgTssXHg== -----END EC PRIVATE KEY-----`

Também é bastante comum ver chaves privadas criptografadas usando uma senha (um segredo compartilhado ou chave simétrica). Eles serão parecidos com isto ( `Proc-Type`e `DEK-Info`fazem parte do PEM e indicam que esta carga útil do PEM é criptografada usando `AES-256-CBC`):

`-----BEGIN EC PRIVATE KEY----- Proc-Type: 4,ENCRYPTED DEK-Info: AES-256-CBC,b3fd6578bf18d12a76c98bda947c4ac9  qdV5u+wrywkbO0Ai8VUuwZO1cqhwsNaDQwTiYUwohvot7Vw851rW/43poPhH07So sdLFVCKPd9v6F9n2dkdWCeeFlI4hfx+EwzXLuaRWg6aoYOj7ucJdkofyRyd4pEt+ Mj60xqLkaRtphh9HWKgaHsdBki68LQbObLOz4c6SyxI= -----END EC PRIVATE KEY-----`

Os objetos PKCS#8 também podem ser criptografados; nesse caso, o rótulo do cabeçalho deve ser "ENCRYPTED PRIVATE KEY" de acordo com RFC 7468. Você não terá cabeçalhos `Proc-Type`e `Dek-Info`neste caso, pois essas informações são codificadas na carga útil.

As chaves públicas geralmente terão uma extensão `.pub`ou . `.pem`As chaves privadas podem conter uma extensão `.prv,` `.key`, ou . `.pem`Mais uma vez, sua milhagem pode variar.

Resumo rápido. ASN.1 é usado para definir tipos de dados como certificados e chaves. DER é um conjunto de regras de codificação para transformar ASN.1 em bits e bytes. X.509 é definido em ASN.1. PKCS#7 e PKCS#12 são estruturas de dados maiores, também definidas usando ASN.1, que podem conter certificados e outras coisas. Eles são comumente usados ​​por Java e Microsoft, respectivamente. Como o DER binário bruto é difícil de desviar pela web, a maioria dos certificados são codificados em PEM, que codifica o DER em base64 e o rotula. As chaves privadas são geralmente representadas como objetos PKCS#8 codificados em PEM. Às vezes, eles também são criptografados com uma senha.

Se isso é confuso, não é você. É o mundo. Tentei.

### INFRAESTRUTURA DE CHAVE PÚBLICA[](https://smallstep.com/blog/everything-pki/#public-key-infrastructure)

É bom saber o que é um certificado, mas isso é menos da metade da história. Vejamos como os certificados são criados e usados.

**Infraestrutura de chave pública** (PKI) é o termo abrangente para tudo o que precisamos para emitir, distribuir, armazenar, usar, verificar, revogar e, de outra forma, gerenciar e interagir com certificados e chaves. É um termo intencionalmente vago, como “infraestrutura de banco de dados”. Os certificados são os blocos de construção da maioria das PKIs, e as autoridades certificadoras são a base. Dito isto, o PKI é muito mais. Inclui bibliotecas, cron jobs, protocolos, convenções, clientes, servidores, pessoas, processos, nomes, mecanismos de descoberta e todas as outras coisas que você precisa para usar a criptografia de chave pública de maneira eficaz.

Se você criar sua própria PKI do zero, terá muita discrição. Assim como se você construísse sua própria infraestrutura de banco de dados. Na verdade, muitas PKIs simples nem usam certificados. Ao editar, `~/.ssh/authorized_keys`você está configurando uma forma simples de PKI sem certificado que o SSH usa para vincular chaves públicas a nomes em arquivos simples. O PGP usa certificados, mas não usa CAs. Em vez disso, usa um modelo [de rede de confiança](https://en.wikipedia.org/wiki/Web_of_trust) . Você pode até [usar um blockchain](http://www.aaronsw.com/weblog/squarezooko) para atribuir nomes e vinculá-los a chaves públicas. A única coisa realmente obrigatória se você estiver construindo uma PKI do zero é que, por definição, você precisa usar chaves públicas. Todo o resto pode mudar.

Dito isto, você provavelmente não deseja construir uma PKI inteiramente do zero. Vamos nos concentrar no tipo de PKI usada na Web e nas PKIs internas que são baseadas em tecnologias de PKI da Web e aproveitam os padrões e componentes existentes.

À medida que avançamos, lembre-se do objetivo simples dos certificados e da PKI: vincular nomes a chaves públicas.

### PKI WEB VS PKI INTERNA[](https://smallstep.com/blog/everything-pki/#web-pki-vs-internal-pki)

Você interage com o Web PKI por meio de seu navegador sempre que acessa um URL HTTPS — como quando carregou este site. Esta é a única PKI com a qual muitas pessoas estão (pelo menos vagamente) familiarizadas. Ele range, faz barulho e tropeça, mas funciona principalmente. Apesar dos seus problemas, melhora substancialmente a segurança na web e é principalmente transparente para os usuários. Você deve usá-lo em todos os lugares em que seu sistema se comunica com o mundo exterior pela Internet.

**O Web PKI** é definido principalmente pela [RFC 5280](https://tools.ietf.org/html/rfc5280) e refinado pelo [CA/Browser Forum](https://cabforum.org/) (também conhecido como CA/B ou CAB Forum). Às vezes é chamado de “Internet PKI” ou PKIX (em homenagem ao grupo de trabalho que o criou). Os documentos do Fórum PKIX e CAB cobrem muito terreno. Eles definem a variedade de certificados de que falamos na última seção. Eles também definem o que é um “nome” e onde ele vai em um certificado, quais algoritmos de assinatura podem ser usados, como uma parte confiável determina o emissor de um certificado, como o período de validade de um certificado (datas de emissão e expiração) é especificado, como funciona a revogação e a validação do caminho do certificado, o processo que as CAs usam para determinar se alguém possui um domínio e muito mais.

O Web PKI é importante porque os certificados Web PKI funcionam por padrão com navegadores e praticamente tudo o mais que usa TLS.

**PKI interna** é a PKI que você mesmo executa, para suas próprias coisas: infraestrutura de produção como serviços, contêineres e VMs; aplicativos de TI corporativos; endpoints corporativos como laptops e telefones; e qualquer outro código ou dispositivo que você queira identificar. Ele permite que você autentique e estabeleça canais criptográficos para que seus itens possam ser executados em qualquer lugar e se comunicar com segurança, mesmo na Internet pública.

Por que executar sua própria PKI interna se a Web PKI já existe? A resposta simples é que o Web PKI não foi projetado para oferecer suporte a casos de uso internos. Mesmo com uma CA como [Let's Encrypt](https://letsencrypt.org/) , que oferece certificados gratuitos e provisionamento automatizado, você terá que lidar com [limites de taxas](https://letsencrypt.org/docs/rate-limits/) e [disponibilidade](https://statusgator.com/services/lets-encrypt) . Isso não é bom se você tiver muitos serviços implantados o tempo todo.

Além disso, com o Web PKI você tem pouco ou nenhum controle sobre detalhes importantes, como vida útil do certificado, mecanismos de revogação, processos de renovação, tipos de chave e algoritmos (todas as coisas importantes que explicaremos em um momento).

Finalmente, os [Requisitos Básicos](https://letsencrypt.org/) do Fórum de CA/Navegador , na verdade, proíbem as CAs de PKI da Web de vincular IPs internos (por exemplo, coisas em `10.0.0.0/8`) ou nomes DNS internos que não sejam totalmente qualificados e resolvíveis em DNS global público (por exemplo, você não pode vincular um Nome DNS do cluster Kubernetes como `foo.ns.svc.cluster.local`). Se quiser vincular esse tipo de nome a um certificado, emitir muitos certificados ou controlar detalhes do certificado, você precisará de sua própria PKI interna.

Na próxima seção veremos que a confiança (ou a falta dela) é mais _um_ motivo para evitar o Web PKI para uso interno. Resumindo, use o Web PKI para seu site público e APIs. Use sua própria PKI interna para todo o resto.

Inscreva-se para receber atualizações

Cancele a assinatura a qualquer momento, consulte [a Política de Privacidade](https://smallstep.com/privacy-policy/)[](https://smallstep.com/privacy-policy/)

### CONFIANÇA E CONFIABILIDADE[](https://smallstep.com/blog/everything-pki/#trust--trustworthiness)

#### Trust Stores

Anteriormente aprendemos a interpretar um certificado como uma declaração, ou afirmação, como: " _o emissor diz que a chave pública do sujeito é blá, blá, blá_ ". Esta reivindicação é assinada pelo emissor para que possa ser autenticada pelas partes confiantes. Omitimos algo importante nesta descrição: "como a parte confiável conhece a chave pública do _emissor ?_

A resposta é simples, se não satisfatória: as partes confiáveis ​​são pré-configuradas com uma lista de **certificados raiz** confiáveis ​​(ou âncoras de confiança) em um **armazenamento confiável** . A maneira como essa pré-configuração ocorre é um aspecto importante de qualquer PKI. Uma opção é inicializar outra PKI: você pode fazer com que alguma ferramenta de automação use SSH para copiar certificados raiz para partes confiáveis, aproveitando a PKI SSH descrita anteriormente. Se você estiver executando na nuvem, seu SSH PKI, por sua vez, será inicializado a partir do Web PKI, além de qualquer autenticação que seu fornecedor de nuvem tenha feito quando você criou sua conta e forneceu seu cartão de crédito. Se você seguir essa _cadeia de confiança_ o suficiente, sempre encontrará pessoas: toda cadeia de confiança termina no espaço de carne.

![2018-12-11-cadeia-de-confiança.jpg](https://smallstep.imgix.net/2018_12_11_chain_of_trust_00c59de759.jpg?auto=format%2Ccompress&fit=max&q=50)

Os certificados raiz em armazenamentos confiáveis ​​são **autoassinados** . O emissor e o assunto são os mesmos. Logicamente, é uma afirmação como " _Mike_ diz que a chave pública de _Mike é_ _blá, blá, blá_ ". A assinatura em um certificado autoassinado fornece garantia de que o sujeito/emissor conhece a chave privada relevante, mas qualquer pessoa pode gerar um certificado autoassinado com qualquer nome que desejar. Portanto, a proveniência é crítica: um certificado autoassinado só deve ser confiável na medida em que o processo pelo qual ele chegou ao armazenamento confiável for confiável. No macOS, o armazenamento confiável é gerenciado pelas chaves. Em muitas distribuições Linux, são simplesmente alguns arquivos dentro `/etc`ou em outro lugar do disco. Se seus usuários puderem modificar esses arquivos, é melhor você confiar em todos os seus usuários.

Então, de onde vêm as lojas confiáveis? Para a Web PKI, as partes confiáveis ​​mais importantes são os navegadores da web. Os armazenamentos confiáveis ​​usados ​​por padrão pelos principais navegadores – e praticamente todo o resto que usa TLS – são mantidos por quatro organizações:

- [Programa de certificado raiz da Apple](http://www.apple.com/certificateauthority/ca_program.html) usado por iOS e macOS
- [Programa de certificado raiz da Microsoft](https://social.technet.microsoft.com/wiki/contents/articles/31633.microsoft-trusted-root-program-requirements.aspx) usado pelo Windows
- [O programa de certificado raiz da Mozilla](https://www.mozilla.org/en-US/about/governance/policies/security-group/certs/) usado por seus produtos e, devido ao seu processo aberto e transparente, usado como base para muitos outros armazenamentos confiáveis ​​(por exemplo, para muitas distribuições Linux)
- [Programa de certificação raiz do Google](https://g.co/chrome/root-policy) usado pelo Chrome em todas as plataformas, exceto iOS.

Os armazenamentos confiáveis ​​do sistema operacional normalmente são fornecidos com o sistema operacional. O Firefox vem com seu próprio armazenamento confiável (distribuído usando TLS de mozilla.org – inicializando a partir do Web PKI usando algum outro armazenamento confiável). Linguagens de programação e outras coisas que não são do navegador, `curl`normalmente usam o armazenamento confiável do sistema operacional por padrão. Portanto, os armazenamentos confiáveis ​​normalmente usados ​​por padrão por praticamente tudo vêm pré-instalados e são atualizados por meio de atualizações de software (que geralmente são assinadas por código usando outra PKI).

Existem mais de 100 autoridades certificadoras comumente incluídas nos armazenamentos confiáveis ​​mantidos por esses programas. Você provavelmente conhece os grandes: Let's Encrypt, Symantec, DigiCert, Entrust, etc. Se você quiser fazer isso de forma programática, o projeto [cfssl](https://github.com/cloudflare/cfssl) da Cloudflare mantém um [repositório no GitHub](https://github.com/cloudflare/cfssl_trust) que inclui os certificados confiáveis ​​de vários armazenamentos confiáveis ​​para ajudar no agrupamento de certificados (que discutiremos em breve). Para uma experiência mais amigável, você pode consultar [o Censys](https://censys.io/) para ver quais certificados são confiáveis ​​para [Mozilla](https://censys.io/certificates?q=validation.nss.valid%3A+true+AND+parsed.extensions.basic_constraints.is_ca%3A+true) , [Apple](https://censys.io/certificates?q=validation.apple.valid%3A+true+AND+parsed.extensions.basic_constraints.is_ca%3A+true) e [Microsoft](https://censys.io/certificates?q=validation.microsoft.valid%3A+true+AND+parsed.extensions.basic_constraints.is_ca%3A+true) .

#### Confiabilidade

Essas mais de 100 autoridades certificadoras são confiáveis ​​no sentido descritivo – navegadores e outras coisas confiam em certificados emitidos por essas CAs por padrão. Mas isso não significa que sejam _confiáveis_ ​​no sentido moral. Pelo contrário, existem casos documentados de autoridades de certificação Web PKI que fornecem aos governos certificados fraudulentos para espionar o tráfego e fazer-se passar por websites. Algumas dessas ACs “confiáveis” operam em jurisdições autoritárias como a China. As democracias também não têm aqui uma base moral elevada. A NSA aproveita todas as oportunidades disponíveis para minar a PKI da Web. Em 2011, as autoridades certificadoras “confiáveis” DigiNotar e Comodo [foram](https://en.wikipedia.org/wiki/DigiNotar) [comprometidas](https://en.wikipedia.org/wiki/Comodo_Group#Certificate_hacking) . A violação do DigiNotar provavelmente foi da NSA. Existem também numerosos exemplos de CAs que emitem por engano certificados malformados ou não conformes. Portanto, embora essas CAs sejam de fato confiáveis, como grupo elas _não_ são empiricamente confiáveis. Em breve veremos que a Web PKI em geral é tão segura quanto a CA menos segura, portanto isso não é bom.

A comunidade de navegadores tomou algumas medidas para resolver esse problema. Os requisitos básicos do CA/Browser Forum racionalizam as regras que essas autoridades de certificação confiáveis ​​devem seguir antes de emitir certificados. As CAs são auditadas quanto à conformidade com essas regras como parte do programa de auditoria WebTrust, que é exigido por alguns programas de certificação raiz para inclusão em seus armazenamentos confiáveis ​​(por exemplo, Mozilla).

Ainda assim, se você estiver usando TLS para coisas internas, provavelmente não vai querer confiar nessas CAs públicas mais do que o necessário. Se fizer isso, provavelmente estará abrindo a porta para a NSA e outros. Você está aceitando o fato de que sua segurança depende da disciplina e dos escrúpulos de mais de 100 outras organizações. Talvez você não se importe, mas um aviso justo.

#### Federação

Para piorar a situação, as partes confiáveis ​​(RPs) da Web PKI confiam em cada CA em seu armazenamento confiável para assinar certificados para qualquer assinante. O resultado é que a segurança geral da Web PKI é tão boa quanto a CA da Web PKI menos segura. O ataque DigiNotar de 2011 demonstrou o problema aqui: como parte do ataque, um certificado foi emitido de forma fraudulenta para google.com. Este certificado era confiável para os principais navegadores da web e sistemas operacionais, apesar do Google não ter nenhum relacionamento com a DigiNotar. Dezenas de outros certificados fraudulentos foram emitidos para empresas como Yahoo!, Mozilla e The Tor Project. Os certificados raiz da DigiNotar foram finalmente removidos dos principais armazenamentos confiáveis, mas é quase certo que muitos danos já haviam sido causados.

Mais recentemente, a Sennheiser [foi chamada por instalar um certificado raiz autoassinado](https://medium.com/asecuritysite-when-bob-met-alice/your-headphones-might-break-the-security-of-your-computer-4f304ed86611) em armazenamentos confiáveis ​​com seu aplicativo HeadSetup e, em seguida, incorporar a chave privada correspondente na configuração do aplicativo. Qualquer pessoa pode extrair esta chave privada e usá-la para emitir um certificado para qualquer domínio. Qualquer computador que tenha o certificado Sennheiser em seu armazenamento confiável confiaria nesses certificados fraudulentos. Isso prejudica completamente o TLS. Ops.

Existem vários mecanismos de mitigação que podem ajudar a reduzir esses riscos. [A Autorização de Autoridade de Certificação](https://tools.ietf.org/html/rfc6844) (CAA) permite restringir quais CAs podem emitir certificados para o seu domínio usando um registro DNS especial. [A Transparência de Certificados](https://www.certificate-transparency.org/) (CT) ( [RFC 6962](https://tools.ietf.org/html/rfc6962) ) exige que as CAs enviem todos os certificados que emitem a um observador imparcial que mantém um [registro de certificado público](https://crt.sh/?Identity=smallstep.com) para detectar certificados emitidos de forma fraudulenta. A prova criptográfica do envio do CT está incluída nos certificados emitidos. [A fixação de chave pública HTTP](https://tools.ietf.org/html/rfc7469) (HPKP ou apenas "fixação") permite que um assinante (um site) diga a um RP (um navegador) para aceitar apenas determinadas chaves públicas em certificados para um domínio específico.

O problema com todas essas coisas é o suporte a RP, ou a falta dele. O CAB Forum agora exige verificações de CAA nos navegadores. Alguns navegadores também oferecem suporte para CT e HPKP. Para outros RPs (por exemplo, a maioria das implementações de biblioteca padrão TLS), esse material quase nunca é aplicado. Este problema surgirá repetidamente: muitas políticas de certificados devem ser aplicadas pelos RPs, e os RPs raramente podem ser incomodados. Se os RPs não verificarem os registros CAA e não exigirem prova de envio de CT, isso não adianta muito.

De qualquer forma, se você executar sua própria PKI interna, deverá manter um armazenamento confiável separado para itens internos. Ou seja, em vez de adicionar seus certificados raiz ao armazenamento confiável do sistema existente, configure solicitações TLS internas para usar apenas suas raízes. Se você deseja uma melhor federação internamente (por exemplo, deseja restringir quais certificados suas CAs internas podem emitir), você pode tentar registros CAA e RPs configurados corretamente. Você também pode conferir [o SPIFFE](https://spiffe.io/) , um esforço de padronização em evolução que aborda esse problema e vários outros relacionados à PKI interna.

### O QUE É UMA AUTORIDADE CERTIFICADORA[](https://smallstep.com/blog/everything-pki/#whats-a-certificate-authority)

Falamos muito sobre autoridades certificadoras (CAs), mas ainda não definimos o que são. Uma CA é um emissor de certificado confiável. Ele atesta a ligação entre uma chave pública e um nome assinando um certificado. Fundamentalmente, uma autoridade de certificação é apenas outro certificado e uma chave privada correspondente usada para assinar outros certificados.

Obviamente, alguma lógica e processo precisam ser envolvidos nesses artefatos. A CA precisa distribuir seu certificado em armazenamentos confiáveis, aceitar e processar solicitações de certificado e emitir certificados para assinantes. Uma CA que expõe APIs acessíveis remotamente para automatizar essas coisas é chamada de _CA online_ . Uma CA com um certificado raiz autoassinado incluído em armazenamentos confiáveis ​​é chamada de _CA raiz_ .

#### Intermediários, cadeias e agrupamento[](https://smallstep.com/blog/everything-pki/#intermediates-chains-and-bundling)

Os [requisitos básicos do CAB Forum](https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-1.6.1.pdf) estipulam que uma chave privada raiz pertencente a uma CA raiz PKI da Web só pode ser usada para assinar um certificado emitindo um comando direto (consulte a seção 4.3.1). Em outras palavras, as CAs raiz da Web PKI não podem automatizar a assinatura de certificados. Eles não podem estar online. Este é um problema para qualquer operação de CA em grande escala. Você não pode fazer com que alguém digite manualmente um comando em uma máquina para atender a todos os pedidos de certificados.

A razão para esta estipulação é a segurança. Os certificados raiz PKI da Web são amplamente distribuídos em armazenamentos confiáveis ​​e difíceis de revogar. Comprometer uma chave privada de CA raiz afetaria literalmente bilhões de pessoas e dispositivos. A melhor prática, portanto, é manter as chaves privadas raiz off-line, de preferência em algum [hardware especializado](https://en.wikipedia.org/wiki/Hardware_security_module) conectado a uma máquina com falta de ar, com boa segurança física e com procedimentos de uso estritamente aplicados.

Muitas PKIs internas também seguem essas mesmas práticas, embora sejam muito menos necessárias. Se você puder automatizar a rotação de certificados raiz (por exemplo, atualizar seus armazenamentos confiáveis ​​usando gerenciamento de configuração ou ferramentas de orquestração), poderá facilmente alternar uma chave raiz comprometida. As pessoas ficam tão obcecadas com o gerenciamento de chaves privadas raiz para PKIs internas que isso atrasa ou impede a implantação de PKI interna. As credenciais da sua conta raiz da AWS são pelo menos tão confidenciais, se não mais. Como você gerencia essas credenciais?

Para tornar a emissão de certificados escalonável (ou seja, para tornar possível a automação) quando a CA raiz não está online, a chave privada raiz é usada raramente para assinar alguns _certificados intermediários_ . As _chaves privadas intermediárias_ correspondentes são usadas por CAs intermediárias (também chamadas de CAs subordinadas) para assinar e emitir certificados folha para assinantes. Os intermediários geralmente não são incluídos em armazenamentos confiáveis, o que os torna mais fáceis de revogar e alternar, portanto, a emissão de certificados de um intermediário normalmente é on-line e automatizada.

Este **pacote** de certificados – folha, intermediário, raiz – forma uma cadeia (chamada _cadeia de certificados_ ). A folha é assinada pelo intermediário, o intermediário é assinado pela raiz e a raiz assina a si mesma.

![2018-12-11-cert-chain.jpg](https://smallstep.imgix.net/2018_12_11_cert_chain_c3b5f291ae.jpg?auto=format%2Ccompress&fit=max&q=50)

Tecnicamente, esta é outra simplificação. Nada impede você de criar cadeias mais longas e gráficos mais complexos (por exemplo, por meio de [certificação cruzada](https://docs.microsoft.com/en-us/windows/desktop/seccertenroll/about-cross-certification) ). Porém, isso geralmente é desencorajado, pois pode se tornar muito complicado muito rapidamente. Em qualquer caso, os certificados da entidade final são nós folha neste gráfico. Daí o nome "certificado folha".

Ao configurar um assinante (por exemplo, um servidor web como Apache ou Nginx ou Linkerd ou Envoy), você normalmente precisará fornecer não apenas o certificado folha, mas um pacote de certificados que inclui intermediários. PKCS#7 e PKCS#12 às vezes são usados ​​aqui porque podem incluir uma cadeia de certificados completa. Mais frequentemente, as cadeias de certificados são codificadas como uma sequência simples de objetos PEM separados por linhas. Algumas coisas esperam que os certificados sejam ordenados de folha a raiz, outras esperam de raiz a folha e algumas coisas não se importam. Inconsistência mais irritante. Google e Stack Overflow ajudam aqui. Ou tentativa e erro.

De qualquer forma, aqui está um exemplo:

`$ cat server.crt -----BEGIN CERTIFICATE----- MIICFDCCAbmgAwIBAgIRANE187UXf5fn5TgXSq65CMQwCgYIKoZIzj0EAwIwHzEd MBsGA1UEAxMUVGVzdCBJbnRlcm1lZGlhdGUgQ0EwHhcNMTgxMjA1MTc0OTQ0WhcN MTgxMjA2MTc0OTQ0WjAUMRIwEAYDVQQDEwlsb2NhbGhvc3QwWTATBgcqhkjOPQIB BggqhkjOPQMBBwNCAAQqE2VPZ+uS5q/XiZd6x6vZSKAYFM4xrYa/ANmXeZ/gh/n0 vhsmXIKNCg6vZh69FCbBMZdYEVOb7BRQIR8Q1qjGo4HgMIHdMA4GA1UdDwEB/wQE AwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwHQYDVR0OBBYEFHee 8N698LZWzJg6SQ9F6/gQBGkmMB8GA1UdIwQYMBaAFAZ0jCINuRtVd6ztucMf8Bun D++sMBQGA1UdEQQNMAuCCWxvY2FsaG9zdDBWBgwrBgEEAYKkZMYoQAEERjBEAgEB BBJtaWtlQHNtYWxsc3RlcC5jb20EK0lxOWItOEdEUWg1SmxZaUJwSTBBRW01eHN5 YzM0d0dNUkJWRXE4ck5pQzQwCgYIKoZIzj0EAwIDSQAwRgIhAPL4SgbHIbLwfRqO HO3iTsozZsCuqA34HMaqXveiEie4AiEAhUjjb7vCGuPpTmn8HenA5hJplr+Ql8s1 d+SmYsT0jDU= -----END CERTIFICATE----- -----BEGIN CERTIFICATE----- MIIBuzCCAWKgAwIBAgIRAKBv/7Xs6GPAK4Y8z4udSbswCgYIKoZIzj0EAwIwFzEV MBMGA1UEAxMMVGVzdCBSb290IENBMB4XDTE4MTIwNTE3MzgzOFoXDTI4MTIwMjE3 MzgzOFowHzEdMBsGA1UEAxMUVGVzdCBJbnRlcm1lZGlhdGUgQ0EwWTATBgcqhkjO PQIBBggqhkjOPQMBBwNCAAT8r2WCVhPGeh2J2EFdmdMQi5YhpMp3hyVZWu6XNDbn xd8QBUNZTHqdsMKDtXoNfmhH//dwz78/kRnbka+acJQ9o4GGMIGDMA4GA1UdDwEB /wQEAwIBpjAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwEgYDVR0TAQH/ BAgwBgEB/wIBADAdBgNVHQ4EFgQUBnSMIg25G1V3rO25wx/wG6cP76wwHwYDVR0j BBgwFoAUcITNjk2XmInW+xfLJjMYVMG7fMswCgYIKoZIzj0EAwIDRwAwRAIgTCgI BRvPAJZb+soYP0tnObqWdplmO+krWmHqCWtK8hcCIHS/es7GBEj3bmGMus+8n4Q1 x8YmK7ASLmSCffCTct9Y -----END CERTIFICATE-----`

Novamente, irritante e barroco, mas não ciência de foguetes.

#### Validação do caminho do certificado[](https://smallstep.com/blog/everything-pki/#certificate-path-validation)

Como os certificados intermediários não estão incluídos em armazenamentos confiáveis, eles precisam ser distribuídos e verificados da mesma forma que os certificados folha. Você fornece esses intermediários ao configurar assinantes, conforme descrito acima. Em seguida, os assinantes os repassam aos RPs. Com o TLS, isso acontece como parte do handshake que estabelece uma conexão TLS. Quando um assinante envia seu certificado para uma parte confiável, ele inclui qualquer intermediário(s) necessário(s) para encadear de volta a uma raiz confiável. A terceira parte confiável verifica os certificados folha e intermediários em um processo denominado **validação do caminho do certificado** .

![2018-12-11-cert-path-validation.jpg](https://smallstep.imgix.net/2018_12_11_cert_path_validation_177f724368.jpg?auto=format%2Ccompress&fit=max&q=50)

O algoritmo completo [de validação do caminho do certificado](https://tools.ietf.org/html/rfc5280#section-6) é complicado. Inclui verificação de expirações de certificados, status de revogação, várias políticas de certificados, restrições de uso de chaves e um monte de outras coisas. A implementação adequada deste algoritmo pelos RPs PKI é absolutamente crítica. As pessoas são surpreendentemente casuais quanto à desativação da validação do caminho do certificado (por exemplo, passando o `-k`sinalizador para `curl`). Não faça isso.

**Não desative a validação do caminho do certificado.** Não é tão difícil fazer o TLS adequado, e a validação do caminho do certificado é a parte do TLS que faz a autenticação. Às vezes as pessoas argumentam que o canal ainda está criptografado, então não importa. Isto é errado. Isto é importante. A criptografia sem autenticação é praticamente inútil. É como um confessionário cego: sua conversa é privada, mas você não tem ideia de quem está do outro lado da cortina. Só que isso não é uma igreja, é a internet. Portanto, **não desative a validação do caminho do certificado** .

### CICLO DE VIDA DA CHAVE E DO CERTIFICADO[](https://smallstep.com/blog/everything-pki/#key--certificate-lifecycle)

Antes de poder usar um certificado com um protocolo como o TLS, você precisa descobrir como obtê-lo de uma CA. Abstratamente, este é um processo bastante simples: um assinante que deseja um certificado gera um par de chaves e envia uma solicitação a uma autoridade certificadora. A CA garante que o nome que será vinculado ao certificado está correto e, se estiver, assina e retorna um certificado.

Os certificados expiram e, nesse momento, não são mais confiáveis ​​pelos RPs. Se você ainda estiver usando um certificado que está prestes a expirar, será necessário renová-lo e alterná-lo. Se você quiser que os RPs parem de confiar em um certificado antes que ele expire, ele poderá (às vezes) ser revogado.

Como grande parte da PKI, esse processo simples é enganosamente complexo. Escondidos nos detalhes estão os dois problemas mais difíceis da ciência da computação: invalidação de cache e nomeação de coisas. Ainda assim, é fácil raciocinar quando você entende o que está acontecendo.

#### Nomeando coisas[](https://smallstep.com/blog/everything-pki/#naming-things)

Historicamente, o X.509 usava _nomes distintos_ (DNs) X.500 para nomear o sujeito de um certificado (um assinante). Um DN inclui um _nome comum_ (para mim, seria "Mike Malone"). Também pode incluir _localidade_ , _país_ , _organização_ , _unidade organizacional_ e um monte de outras porcarias irrelevantes (lembre-se de que esse material foi originalmente criado para uma lista telefônica digital). Ninguém entende nomes distintos. Eles realmente não fazem sentido para a web. Evite-os. Se você usá-los, mantenha-os simples. Você não precisa usar todos os campos. Na verdade, você _não deveria_ . Um nome comum é provavelmente tudo que você precisa, e talvez um nome de organização, se você gosta de emoções fortes.

O PKIX especificou originalmente que o nome do host DNS de um site deveria ser vinculado ao _nome comum_ do DN . Mais recentemente, o Fórum CAB descontinuou esta prática e tornou todo o DN opcional (ver seções 7.1.4.2 dos [Requisitos de Linha de Base](https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-1.6.1.pdf) ). Em vez disso, as práticas recomendadas modernas são aproveitar a [extensão X.509 do nome alternativo da entidade (SAN)](https://tools.ietf.org/html/rfc5280#section-4.2.1.6) para vincular um nome em um certificado.

Existem quatro tipos de SANs de uso comum, todos os quais vinculam nomes amplamente usados ​​e compreendidos: nomes de domínio (DNS), endereços de e-mail, endereços IP e URIs. Eles já deveriam ser únicos nos contextos nos quais estamos interessados ​​e mapeiam muito bem as coisas que estamos interessados ​​em identificar: endereços de e-mail para pessoas, nomes de domínio e endereços IP para máquinas e códigos, URIs, se você quiser para ficar chique. Use SANs.

![2018-12-11-inspecione-san-dns.jpg](https://smallstep.imgix.net/2018_12_11_inspect_san_dns_c4cac54b96.jpg?auto=format%2Ccompress&fit=max&q=50)

Observe também que o Web PKI permite que vários nomes sejam vinculados a um certificado e permite caracteres curinga nos nomes. Um certificado pode ter vários SANs e SANs como `*.smallstep.com`. Isto é útil para sites que respondem a vários nomes (por exemplo, `smallstep.com`e `www.smallstep.com`).

#### Gerando pares de chaves[](https://smallstep.com/blog/everything-pki/#generating-key-pairs)

Assim que tivermos um nome, precisamos gerar um par de chaves antes de podermos criar um certificado. Lembre-se de que a segurança de uma PKI depende criticamente de uma invariante simples: a única entidade que conhece uma determinada chave privada é o assinante nomeado no certificado correspondente. Para ter certeza de que essa invariante é válida, a melhor prática é fazer com que o assinante gere seu próprio par de chaves para que seja a única coisa que o _conheça_ . Definitivamente, evite transmitir uma chave privada pela rede.

Você precisará decidir que tipo de chave deseja usar. Essa é outra postagem, mas aqui estão algumas orientações rápidas (em maio de 2023). Há uma transição lenta, mas contínua, de RSA para chaves de curva elíptica ( [ECDSA](https://blog.cloudflare.com/ecdsa-the-digital-signature-algorithm-of-a-better-internet/) ou [EdDSA](https://tools.ietf.org/html/rfc8032) ). Se você decidir usar chaves RSA, faça-as com pelo menos 2.048 bits e não se preocupe com nada maior que 4.096 bits. E use RSA-PSS, não RSA PKCS#1. Se você usa ECDSA, a curva P-256 é provavelmente a melhor ( `secp256k1`ou `prime256v1`em openssl)... a menos que você esteja preocupado com a NSA, caso em que você pode optar por usar algo mais sofisticado como EdDSA com Curve25519 (embora o suporte para essas chaves seja nada bom).

Aqui está um exemplo de geração de um par de chaves P-256 de curva elíptica usando `openssl`:

`openssl ecparam -name prime256v1 -genkey -out k.prv openssl ec -in k.prv -pubout -out k.pub`

Aqui está um exemplo de geração do mesmo tipo de par de chaves usando `step`:

`step crypto keypair --kty EC --curve P-256 k.pub k.prv`

Você também pode fazer isso programaticamente e nunca permitir que suas chaves privadas toquem no disco.

Escolha o seu veneno.

#### Emissão[](https://smallstep.com/blog/everything-pki/#issuance)

Depois que o assinante tiver um nome e um par de chaves, a próxima etapa é obter um certificado folha de uma CA. A CA vai querer autenticar (provar) duas coisas:

- A chave pública a ser vinculada ao certificado é a chave pública do assinante (ou seja, o assinante conhece a chave privada correspondente)
- O nome a ser vinculado no certificado é o nome do assinante

O primeiro é normalmente conseguido através de um mecanismo técnico simples: uma solicitação de assinatura de certificado. Este último é mais difícil. Abstratamente, o processo é denominado prova de identidade ou registro.

##### Solicitações de assinatura de certificado[](https://smallstep.com/blog/everything-pki/#certificate-signing-requests)

Para solicitar um certificado, um assinante envia uma _solicitação de assinatura de certificado_ (CSR) a uma autoridade de certificação. O CSR é outra estrutura ASN.1, definida por [PKCS#10](https://tools.ietf.org/html/rfc2986) .

Assim como um certificado, um CSR é uma estrutura de dados que contém uma chave pública, um nome e uma assinatura. É autoassinado usando a chave privada que corresponde à chave pública no CSR. Esta assinatura prova que tudo o que criou o CSR conhece a chave privada. Também permite que o CSR seja copiado e colado e desviado sem a possibilidade de modificação por algum intruso.

Os CSRs incluem muitas opções para especificar detalhes do certificado. Na prática, a maior parte dessas coisas é ignorada pelas CAs. Em vez disso, a maioria das ACs utiliza um modelo ou fornece uma interface administrativa para coletar essas informações.

Você pode gerar um par de chaves e criar um CSR usando `step`um comando como este:

passo certificado criar --csr test.smallstep.com test.csr test.key

OpenSSL é super poderoso, mas [muito mais irritante](https://www.openssl.org/docs/manmaster/man1/openssl-req.html) .

##### Prova de identidade[](https://smallstep.com/blog/everything-pki/#identity-proofing)

Depois que uma CA recebe um CSR e verifica sua assinatura, a próxima coisa que precisa fazer é descobrir se o nome a ser vinculado no certificado é realmente o nome correto do assinante. Isso é complicado. O objetivo dos certificados é permitir que os RPs autentiquem assinantes, mas como a CA deve autenticar o assinante antes que um certificado seja emitido?

A resposta é: depende. Para o Web PKI existem três tipos de certificados e as maiores diferenças são como eles identificam os assinantes e o tipo de prova de identidade empregada. São eles: certificados de validação de domínio (DV), validação de organização (OV) e validação estendida (EV).

Os certificados DV vinculam um nome DNS e são emitidos com base na prova de controle sobre um nome de domínio. A revisão normalmente ocorre por meio de uma cerimônia simples, como o envio de um e-mail de confirmação ao contato administrativo listado nos registros WHOIS. O [protocolo ACME](https://www.openssl.org/docs/manmaster/man1/openssl-req.html) , originalmente desenvolvido e usado pela Let's Encrypt, melhora esse processo com melhor automação: em vez de usar verificação de e-mail, uma CA ACME emite um desafio que o assinante deve completar para provar que controla um domínio. A parte do desafio da especificação ACME é um ponto de extensão, mas os desafios comuns incluem servir um número aleatório em uma determinada URL (o desafio HTTP) e colocar um número aleatório em um registro DNS TXT (o desafio DNS).

Os certificados OV e EV baseiam-se em certificados DV e incluem o nome e a localização da organização proprietária do nome de domínio vinculado. Eles conectam um certificado não apenas a um nome de domínio, mas também à entidade legal que o controla. O processo de verificação para certificados OV não é consistente entre CAs. Para resolver isso, o CAB Forum introduziu [certificados EV](https://cabforum.org/wp-content/uploads/CA-Browser-Forum-EV-Guidelines-1.8.0.pdf) . Eles incluem as mesmas informações básicas, mas exigem requisitos rigorosos de verificação (prova de identidade). O processo EV pode levar dias ou semanas e pode incluir pesquisas em registros públicos e atestados (em papel) assinados por dirigentes corporativos (com canetas). E no final das contas, os navegadores da web não diferenciam de forma alguma os certificados EV de forma proeminente. Portanto, os certificados EV não são amplamente aproveitados pelas partes confiáveis ​​da Web PKI.

Essencialmente, todo Web PKI RP requer apenas garantia de nível DV, com base na “prova” de controle de um domínio. É importante considerar o que exatamente um certificado DV _realmente_ prova. Deve _provar_ que a entidade que solicita o certificado possui o domínio relevante. _Na verdade,_ prova que, em algum momento, a entidade que solicitou o certificado foi capaz de ler um e-mail _ou_ configurar DNS _ou_ fornecer um segredo via HTTP. A segurança subjacente de DNS, email e BGP da qual esses processos dependem não é ótima. Os ataques contra esta infra-estrutura [têm ocorrido](https://doublepulsar.com/hijack-of-amazons-internet-domain-service-used-to-reroute-web-traffic-for-two-hours-unnoticed-3a6f0dda6a6f) com a intenção de obter certificados fraudulentos.

Para PKI interna, você pode usar qualquer processo que desejar para prova de identidade. Você provavelmente pode fazer melhor do que depender de DNS ou e-mail como o Web PKI faz. Isso pode parecer difícil no início, mas na verdade não é. Você pode aproveitar a infraestrutura confiável existente: tudo o que você usa para provisionar seus itens também deve ser capaz de medir e atestar a identidade de tudo o que está sendo provisionado. Se você confia no Chef ou no Puppet ou no Ansible ou no Kubernetes para colocar código nos servidores, você pode confiar neles para atestados de identidade. Se estiver usando AMIs brutas na AWS, você poderá usar [documentos de identidade de instância](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html) ( [GCP](https://cloud.google.com/compute/docs/instances/verifying-instance-identity) e [Azure](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-use-vm-token) têm funcionalidade semelhante).

Sua infraestrutura de provisionamento _deve_ ter alguma noção de identidade para poder colocar o código certo no lugar certo e iniciar tudo. E você _deve_ confiar nisso. Você pode aproveitar esse conhecimento e confiança para configurar armazenamentos confiáveis ​​de RP e inicializar assinantes em sua PKI interna. Tudo o que você precisa fazer é encontrar uma maneira de sua infraestrutura de provisionamento informar à sua CA a identidade do que está sendo iniciado. Aliás, esta é precisamente a lacuna que [os certificados de etapa](https://smallstep.com/certificates/) foram projetados para preencher.

#### Expiração[](https://smallstep.com/blog/everything-pki/#expiration)

Os certificados expiram… normalmente. Este não é um requisito estrito por si só, mas quase sempre é verdade. Incluir uma expiração num certificado é importante porque a utilização do certificado é desagregada: em geral, não há nenhuma autoridade central que seja interrogada quando um certificado é verificado por um RP. Sem uma data de validade, os certificados seriam confiáveis ​​para sempre. Uma regra prática para segurança é que, à medida que nos aproximamos do fim, a probabilidade de uma credencial ser comprometida se aproxima de 100%. Assim, os certificados expiram.

Em particular, os certificados X.509 incluem um período de validade: _emitido no_ momento, _não antes_ e _não depois_ . O tempo avança, eventualmente passa do tempo _não tardio_ e o certificado morre. Essa inevitabilidade aparentemente inócua tem algumas sutilezas importantes.

Primeiro, não há nada que impeça um RP específico de aceitar um certificado expirado por engano (ou design incorreto). Novamente, o uso do certificado é desagregado. Cabe a cada RP verificar se um certificado expirou e, às vezes, eles erram. Isso pode acontecer se o seu código depender de um relógio do sistema que não esteja sincronizado corretamente. Um cenário comum é um sistema cujo relógio é redefinido para a época unix que não confia em nenhum certificado porque pensa que é 1º de janeiro de 1970 - bem antes do horário _não anterior_ em qualquer certificado emitido recentemente. Portanto, certifique-se de que seus relógios estejam sincronizados!

Do lado do assinante, o material da chave privada precisa ser tratado adequadamente após a expiração do certificado. Se um par de chaves foi usado para assinatura/autenticação (por exemplo, com TLS), você desejará excluir a chave privada quando ela não for mais necessária. Manter uma chave de assinatura por perto é um risco de segurança desnecessário: ela não serve para nada além de assinaturas fraudulentas. No entanto, se o seu par de chaves foi usado para criptografia, a situação é diferente. Você precisará manter a chave privada enquanto ainda houver dados criptografados na chave. Se alguma vez lhe disseram para não usar o mesmo par de chaves para assinatura e criptografia, este é o principal motivo. Usar a mesma chave para assinatura e criptografia torna impossível implementar as melhores práticas de gerenciamento do ciclo de vida de chaves quando uma chave privada não é mais necessária para assinatura: força você a continuar assinando chaves por mais tempo do que o necessário se ainda for necessário para descriptografar coisas.

#### Renovação[](https://smallstep.com/blog/everything-pki/#renewal)

Se você ainda estiver usando um certificado que está prestes a expirar, você vai querer renová-lo antes que isso aconteça. Na verdade, não existe um processo de renovação padrão para Web PKI – não existe uma maneira formal de estender o período de validade de um certificado. Em vez disso, basta substituir o certificado expirado por um novo. Portanto, o processo de renovação é igual ao processo de emissão: gerar e enviar um CSR e cumprir quaisquer obrigações de comprovação de identidade.

Para PKI interna podemos fazer melhor. A coisa mais fácil a fazer é usar seu certificado antigo com um protocolo como o TLS mútuo para renovar. A CA pode autenticar o certificado de cliente apresentado pelo assinante, assiná-lo novamente com uma expiração estendida e retornar o novo certificado em resposta. Isso torna a renovação automatizada muito fácil e ainda força os assinantes a verificarem-se periodicamente com uma autoridade central. Você pode usar esse processo de check-in para criar facilmente recursos de monitoramento e revogação.

Em ambos os casos, a parte mais difícil é simplesmente lembrar-se de renovar os seus certificados antes que expirem. Quase todo mundo que gerencia certificados para um site público teve um deles expirado inesperadamente, produzindo um erro [como este](https://expired.badssl.com/) . Meu melhor conselho aqui é: se algo dói, faça mais. Use certificados de curta duração. Isso o forçará a melhorar seus processos e automatizar esse problema. Let's Encrypt facilita a automação e emite certificados de 90 dias, o que é muito bom para Web PKI. Para PKI interna, você provavelmente deveria usar ainda menos tempo: vinte e quatro horas ou menos. Existem alguns desafios de implementação – [a rotação de certificados sem ocorrência](https://diogomonica.com/2017/01/11/hitless-tls-certificate-rotation-in-go/) pode ser um pouco complicada – mas vale a pena o esforço.

Dica rápida que você pode usar `step`para verificar o tempo de expiração de um certificado na linha de comando:

`step certificate inspect cert.pem --format json | jq .validity.end step certificate inspect https://smallstep.com --format json | jq .validity.end`

É uma coisa pequena, mas se você combinar isso com uma transferência de zona DNS em um pequeno script bash, poderá obter um monitoramento decente em torno da expiração do certificado para todos os seus domínios para ajudar a detectar problemas antes que eles se tornem interrupções.

#### Revogação[](https://smallstep.com/blog/everything-pki/#revocation)

Se uma chave privada estiver comprometida ou um certificado simplesmente não for mais necessário, você poderá revogá-lo. Ou seja, você pode querer marcá-lo ativamente como inválido para que ele deixe de ser confiável pelos RPs imediatamente, mesmo antes de expirar. Revogar certificados X.509 é [uma](https://maikel.pro/blog/current-state-certificate-revocation-crls-ocsp/) [grande](https://scotthelme.co.uk/revocation-is-broken/) [bagunça](https://www.imperialviolet.org/2014/04/19/revchecking.html) . Assim como a expiração, a responsabilidade recai sobre os RPs para fazer cumprir as revogações. Ao contrário da expiração, o estado de revogação não pode ser codificado no certificado. O RP precisa determinar o status de revogação do certificado por meio de algum processo fora de banda. A menos que seja explicitamente configurado, [a maioria](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=745837) [dos RPs](https://stackoverflow.com/questions/16244084/how-to-programmatically-check-if-a-certificate-has-been-revoked) [TLS](https://stackoverflow.com/questions/39297240/python-failed-to-verify-any-crls-for-ssl-tls-connections?rq=1) [da Web](https://github.com/golang/go/issues/18323) [PKI](https://stackoverflow.com/questions/38301283/java-ssl-certificate-revocation-checking) [não](https://github.com/nodejs/node/issues/16338) [se incomodam](https://forums.developer.apple.com/thread/24298) . Em outras palavras, por padrão, a maioria das implementações de TLS aceitará certificados revogados.[](https://stackoverflow.com/questions/39297240/python-failed-to-verify-any-crls-for-ssl-tls-connections?rq=1)[](https://stackoverflow.com/questions/16244084/how-to-programmatically-check-if-a-certificate-has-been-revoked)[](https://github.com/nodejs/node/issues/16338)[](https://forums.developer.apple.com/thread/24298)

Para a PKI interna a tendência é aceitar esta realidade e utilizar _a revogação passiva_ . Ou seja, emitir certificados que expiram com rapidez suficiente para que a revogação não seja necessária. Se quiser “revogar” um certificado, você simplesmente desativa a renovação e espera que ele expire. Para que isso funcione, você precisa usar certificados de curta duração. Quão curto? Isso depende do seu modelo de ameaça (é como dizem os profissionais de segurança ¯\ _(ツ)_ /¯). Vinte e quatro horas é bastante típico, mas também expirações muito mais curtas, como cinco minutos. Existem desafios óbvios em torno da escalabilidade e da disponibilidade se você encurtar muito a vida útil: cada renovação requer interação com uma CA on-line, portanto, é melhor que sua infraestrutura de CA seja escalável e altamente disponível. À medida que você diminui a vida útil do certificado, lembre-se de manter todos os seus relógios sincronizados ou você terá problemas.

Para a Web e outros cenários onde a revogação passiva não funciona, a primeira coisa que você deve fazer é parar e reconsiderar a revogação passiva. Se você _realmente_ precisa de uma revogação, você tem duas opções:

- Listas de revogação de certificados (CRLs)
- Protocolo de assinatura de certificado online (OCSP)

As CRLs são definidas junto com um milhão de outras coisas na RFC 5280. Elas são simplesmente uma lista assinada de números de série que identificam certificados revogados. A lista é servida a partir de um _ponto de distribuição de CRL_ : uma URL incluída no certificado. A expectativa é que as partes confiantes baixem essa lista e a questionem quanto ao status de revogação sempre que verificarem um certificado. Existem alguns problemas óbvios aqui: as CRLs podem ser grandes e os pontos de distribuição podem cair. Se os RPs verificarem as CRLs, eles armazenarão em cache a resposta do ponto de distribuição e sincronizarão apenas periodicamente. Na web, as CRLs costumam ser armazenadas em cache por dias. Se demorar tanto para que as CRLs se propaguem, você também pode usar a revogação passiva. Também é comum que os RPs _falhem ao abrir_ – para aceitar um certificado se o ponto de distribuição da CRL estiver inativo. Isso pode ser um problema de segurança: você pode enganar um RP para que aceite um certificado revogado montando um ataque de negação de serviço contra o ponto de distribuição da CRL.

Pelo que vale a pena, mesmo se você estiver usando CRLs, você deve considerar o uso de certificados de curta duração para manter o tamanho da CRL baixo. A CRL só precisa incluir números de série para certificados que foram revogados _e_ ainda não expiraram. Se seus certificados tiverem vida útil mais curta, suas CRLs serão mais curtas.

Se você não gosta de CRL, sua outra opção é [OCSP](https://www.ietf.org/rfc/rfc2560.txt) , que permite que RPs consultem um _respondente OCSP_ com um número de série de certificado para obter o status de revogação de um certificado específico. Tal como o ponto de distribuição da CRL, o URL do respondedor OCSP está incluído no certificado. OCSP parece legal (e óbvio), mas tem seus próprios problemas. Isso levanta sérios problemas de privacidade para o Web PKI: o respondente do OCSP pode ver quais sites estou visitando com base nas verificações de status do certificado que enviei. Também adiciona sobrecarga a cada conexão TLS: uma solicitação adicional deve ser feita para verificar o status de revogação. Assim como a CRL, muitos RPs (incluindo navegadores) falham ao abrir e assumem que um certificado é válido se o respondedor OCSP estiver inativo ou retornar um erro.

O grampeamento OCSP é uma variante do OCSP que supostamente corrige esses problemas. Em vez de a parte confiável acessar o respondedor OCSP, o assinante que possui o certificado o faz. A resposta OCSP é um atestado assinado com um curto prazo de validade informando que o certificado não foi revogado. O atestado está incluído no handshake TLS ("grampeado" no certificado) entre o assinante e o RP. Isso fornece ao RP um status de revogação razoavelmente atualizado, sem a necessidade de consultar diretamente o respondente do OCSP. O assinante pode usar uma resposta OCSP assinada diversas vezes, até que ela expire. Isso reduz a carga do respondente, elimina principalmente problemas de desempenho e resolve o problema de privacidade com OCSP. No entanto, tudo isso é um dispositivo um tanto quanto rube goldberg. Se os assinantes estão recorrendo a alguma autoridade para obter um atestado assinado de curta duração informando que um certificado não expirou, por que não eliminar o intermediário: basta usar certificados de curta duração.

### USANDO CERTIFICADOS[](https://smallstep.com/blog/everything-pki/#using-certificates)

Com todo esse histórico resolvido, _usar_ certificados é realmente fácil. Demonstraremos com TLS, mas a maioria dos outros usos são bastante semelhantes.

- Para configurar uma parte confiável de PKI, você informa quais certificados raiz usar
- Para configurar um assinante PKI, você informa qual certificado e chave privada usar (ou como gerar seu próprio par de chaves e trocar um CSR por um certificado em si)

É muito comum que uma entidade (código, dispositivo, servidor, etc.) seja um RP e um assinante. Essas entidades precisarão ser configuradas com o(s) certificado(s) raiz e um certificado e uma chave privada. Finalmente, para Web PKI, os certificados raiz corretos geralmente são confiáveis ​​por padrão, portanto você pode pular essa parte.

Aqui está um exemplo completo que demonstra a emissão de certificado, distribuição de certificado raiz e configuração de cliente (RP) e servidor (assinante) TLS:

![2018-12-11-step-ca-certificate-flow.jpg](https://smallstep.imgix.net/2018_12_11_step_ca_certificate_flow_c8589cfcac.jpg?auto=format%2Ccompress&fit=max&q=50)

Esperamos que isso ilustre como a PKI e o TLS internos corretos e corretos podem ser simples. Você não precisa usar certificados autoassinados ou fazer coisas perigosas, como desabilitar a validação do caminho do certificado (passar `-k`para `curl`).

Praticamente todos os clientes e servidores TLS usam esses mesmos parâmetros. Quase todos eles apostam na parte do ciclo de vida da chave e do certificado: eles geralmente assumem que os certificados aparecem magicamente no disco, são girados, etc. Essa é a parte difícil. Novamente, se você precisar disso, é isso que os certificados de etapa fazem.

## Resumindo[](https://smallstep.com/blog/everything-pki/#in-summary)

A criptografia de chave pública permite que os computadores “vejam” as redes. Se eu tiver uma chave pública, posso "ver" que você tem a chave privada correspondente, mas não posso usá-la sozinho. Se eu não tiver sua chave pública, os certificados podem ajudar. Os certificados vinculam as chaves públicas ao nome do proprietário da chave privada correspondente. São como carteiras de motorista para computadores e códigos. As autoridades certificadoras (CAs) assinam certificados com suas chaves privadas, atestando essas ligações. Eles são como o DMV. Se você for o único que se parece com você e me mostrar uma carteira de motorista de um DMV em que confio, posso descobrir seu nome. Se você é o único que conhece uma chave privada e me envia um certificado de uma CA em que confio, posso descobrir seu nome.

No mundo real, a maioria dos certificados são certificados X.509 v3. Eles são definidos usando ASN.1 e geralmente serializados como DER codificado em PEM. As chaves privadas correspondentes são geralmente representadas como objetos PKCS#8, também serializados como DER codificado em PEM. Se você usa Java ou Microsoft, poderá executar os formatos de envelope PKCS#7 e PKCS#12. Há muita bagagem histórica aqui que pode tornar esse trabalho bastante frustrante, mas é mais irritante do que difícil.

Infraestrutura de chave pública é o termo abrangente para todas as coisas que você precisa construir e concordar para usar chaves públicas de maneira eficaz: nomes, tipos de chave, certificados, CAs, cron jobs, bibliotecas, etc. Web PKI é a PKI pública usada por padrão pelos navegadores da web e praticamente tudo o mais que usa TLS. As CAs da Web PKI são confiáveis, mas não confiáveis. A PKI interna é a sua própria PKI que você mesmo cria e executa. Você quer um porque o Web PKI não foi projetado para casos de uso internos e porque o PKI interno é mais fácil de automatizar, mais fácil de escalar e oferece mais controle sobre muitas coisas importantes, como nomenclatura e vida útil do certificado. Use Web PKI para coisas públicas. Use sua própria PKI interna para coisas internas (por exemplo, para [usar TLS](https://smallstep.com/blog/use-tls.html) para substituir VPNs). [O Smallstep Certificate Manager](https://smallstep.com/certificate-manager) torna a construção de uma PKI interna muito fácil.

Para obter um certificado, você precisa nomear coisas e gerar chaves. Use SANs para nomes: SANs DNS para códigos e máquinas, SANs EMAIL para pessoas. Use SANs URI se não funcionarem. O tipo de chave é um grande tópico que não tem importância: você pode alterar os tipos de chave e a criptografia real não será o elo mais fraco em sua PKI. Para obter um certificado de uma CA, você envia um CSR e prova sua identidade. Use certificados de curta duração e revogação passiva. Automatize a renovação de certificados. Não desative a validação do caminho do certificado.

Lembre-se: certificados e PKI vinculam nomes a chaves públicas. O resto são só detalhes.