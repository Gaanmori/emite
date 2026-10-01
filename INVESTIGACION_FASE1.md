# Fase 1 — Investigación técnica: hallazgos

Notas de trabajo de la Fase 1 (sección 7/8 del [README](README.md)). Fuente: los
Anexos Técnicos oficiales de la DIAN, descargados y procesados directamente
(`pdftotext`) para poder buscar en el texto completo en vez de hojear PDFs de
cientos de páginas:

- **Factura Electrónica de Venta — Anexo Técnico v1.9** (Resolución 000165 de 2023),
  753 páginas:
  https://www.dian.gov.co/impuestos/factura-electronica/Documents/Anexo-Tecnico-Factura-Electronica-de-Venta-vr-1-9.pdf
- **Documento Equivalente Electrónico — Anexo Técnico v1.0** (1546 páginas):
  https://www.dian.gov.co/impuestos/factura-electronica/Documents/Anexo-Tecnico-Documento-Equivalente-Electronico-V1-0-final.pdf
  — **ojo:** este documento cubre *11 tipos distintos* de documento equivalente
  (tiquete POS, servicios públicos, peajes, transporte terrestre, tiquete aéreo,
  juegos localizados, boleta de espectáculos, bolsa de valores, boleta de cine,
  extracto, y notas de ajuste). El caso de Dorato — y el que le interesa a
  `emite` para v1 — es específicamente la **sección 8.2: "Documento equivalente
  electrónico tiquete de máquina registradora con sistema P.O.S."** Todo lo de
  abajo sobre "DEE POS" se refiere a esa sección, no al documento completo.
- También revisado: [dian-kit](https://github.com/sergioarojasm98/dian-kit), un SDK
  open-source (MIT, TypeScript/Node.js) que ya implementa esto y dice estar validado
  contra el sandbox real de la DIAN — útil como referencia de implementación, no
  como fuente normativa.

Todo lo de abajo está citado con su página del PDF para poder volver a verificarlo.

---

## 1. Algoritmo del CUFE (factura electrónica) — verificado byte a byte

**Fórmula** (Anexo Técnico Factura, pág. 655):

```
CUFE = SHA-384(NumFac + FecFac + HorFac + ValFac + CodImp1 + ValImp1 + CodImp2 +
               ValImp2 + CodImp3 + ValImp3 + ValTot + NitOFE + NumAdq + ClTec +
               TipoAmbiente)
```

`+` significa concatenación directa de texto, sin separadores.

| Campo | Significado | Formato |
|---|---|---|
| `NumFac` | Prefijo + número de la factura | texto |
| `FecFac` | Fecha de factura | `YYYY-MM-DD` |
| `HorFac` | Hora de factura, con zona horaria | `hh:mm:ss-05:00` |
| `ValFac` | Valor de la factura sin impuestos | decimal, 2 dígitos, truncado (no redondeado), sin separador de miles |
| `CodImp1` | Fijo: `"01"` (IVA) | — |
| `ValImp1` | Valor del IVA (0.00 si no aplica) | igual formato que ValFac |
| `CodImp2` | Fijo: `"04"` (INC) | — |
| `ValImp2` | Valor del Impuesto Nacional al Consumo (0.00 si no aplica) | igual |
| `CodImp3` | Fijo: `"03"` (ICA) | — |
| `ValImp3` | Valor del ICA (0.00 si no aplica) | igual |
| `ValTot` | Valor total a pagar | igual |
| `NitOFE` | NIT del facturador, sin puntos/guiones/DV | texto |
| `NumAdq` | NIT/identificación del adquiriente, sin puntos/guiones/DV | texto |
| `ClTec` | Clave técnica del rango de numeración autorizado — **no viene en el XML**, se obtiene al consultar el rango vía webservice (`GetNumberingRange`) | texto |
| `TipoAmbiente` | `1` = producción, `2` = habilitación/pruebas | `/Invoice/cbc:ProfileExecutionID` |

**Ejemplo oficial, verificado computacionalmente** (pág. 656-657):

```
NumFac=323200000129  FecFac=2019-01-16  HorFac=10:53:10-05:00  ValFac=1500000.00
CodImp1=01  ValImp1=285000.00  CodImp2=04  ValImp2=0.00  CodImp3=03  ValImp3=0.00
ValTot=1785000.00  NitOFE=700085371  NumAdq=800199436
ClTec=693ff6f2a553c3646a063436fd4dd9ded0311471  TipoAmbiente=1

CUFE = SHA384(
  "323200000129" + "2019-01-16" + "10:53:10-05:00" + "1500000.00" + "01" +
  "285000.00" + "04" + "0.00" + "03" + "0.00" + "1785000.00" + "700085371" +
  "800199436" + "693ff6f2a553c3646a063436fd4dd9ded0311471" + "1"
)
= 8bb918b19ba22a694f1da11c643b5e9de39adf60311cf179179e9b33381030bcd4c3c3f156c506ed5908f9276f5bd9b4
```

Corrí este cálculo en Python (`hashlib.sha384`) y coincide exactamente con el
resultado del documento. **El algoritmo está confirmado, no es una interpretación.**

### XPath de cada campo en el XML (Invoice)

```
NumFac   /Invoice/cbc:ID
FecFac   /Invoice/cbc:IssueDate
HorFac   /Invoice/cbc:IssueTime
ValFac   /Invoice/cac:LegalMonetaryTotal/cbc:LineExtensionAmount
CodImp1/ValImp1  /Invoice/cac:TaxTotal[x]/cac:TaxSubtotal/cac:TaxCategory/cac:TaxScheme/cbc:ID = 01  →  .../cbc:TaxAmount
CodImp2/ValImp2  ídem con ID = 04
CodImp3/ValImp3  ídem con ID = 03
ValTot   /Invoice/cac:LegalMonetaryTotal/cbc:PayableAmount
NitOFE   /Invoice/cac:AccountingSupplierParty/cac:Party/cac:PartyTaxScheme/cbc:CompanyID
NumAdq   /Invoice/cac:AccountingCustomerParty/cac:Party/cac:PartyTaxScheme/cbc:CompanyID
ClTec    NO está en el XML — sale de la consulta de rango de numeración (GetNumberingRange)
TipoAmbiente  /Invoice/cbc:ProfileExecutionID
```

## 2. Algoritmo del CUDE — confirmado para DEE POS específicamente, con una diferencia clave frente al CUFE

El CUDE usa el mismo patrón (`SHA-384` sobre concatenación de campos), pero **con
una diferencia real frente al CUFE de factura**, confirmada en el propio Anexo
Técnico del DEE POS (pág. 1420-1431):

```
CUDE = SHA-384(NumFac + FecFac + HorFac + ValFac + CodImp1 + ValImp1 + CodImp2 +
               ValImp2 + CodImp3 + ValImp3 + ValTot + NitOFE + NumAdq +
               Software-PIN + TipoAmbiente)
```

**La diferencia:** donde el CUFE de factura usa `ClTec` (la clave técnica del rango
de numeración, obtenida vía `GetNumberingRange`), el CUDE del DEE POS usa
**`Software-PIN`** — un PIN distinto, asignado al registrar el software en el
"Catálogo-DIAN" del participante, marcado explícitamente como *"valor reservado, de
circulación restringida"*. Ninguno de los dos (`ClTec` ni `Software-PIN`) viaja
dentro del XML — ambos son secretos que `emite` tiene que guardar por tenant junto
al certificado (sección 4.1 del README), pero son dos secretos *distintos* según el
tipo de documento que se esté emitiendo.

Aplica igual a las notas de ajuste (`/CreditNote/...`, `/DebitNote/...`) y al
`ApplicationResponse`.

**Ejemplo oficial, verificado computacionalmente** (pág. 1426, nota de ajuste —
resulta ser el mismo ejemplo numérico que el Anexo de Factura usa para su nota
crédito, así que ya estaba verificado sin saberlo):

```
NumCr=8110007871  FecCr=2019-01-12  HorCr=07:00:00-05:00  VaCRL=5000.00
CodImp1=01  ValImp1=950.00  CodImp2=04  ValImp2=0.00  CodImp3=03  ValImp3=0.00
ValTot=5950.00  NitOFE=900373076  NumAdq=8355990  Software-PIN=12301  TipoAmbiente=1

CUDE = SHA384(
  "8110007871" + "2019-01-12" + "07:00:00-05:00" + "5000.00" + "01" + "950.00" +
  "04" + "0.00" + "03" + "0.00" + "5950.00" + "900373076" + "8355990" + "12301" + "1"
)
= 907e4444decc9e59c160a2fb3b6659b33dc5b632a5008922b9a62f83f757b1c448e47f5867f2b50dbdb96f48c7681168
```

Corrí este también en Python — coincide exactamente.

**El atributo `@schemeName`** de la etiqueta `cbc:UUID` debe ser literalmente
`"CUFE-SHA384"` o `"CUDE-SHA384"` (pág. 683 y otras) — la DIAN lo valida.

## 3. Firma digital: es XAdES-**EPES**, no XAdES-BES (corrección al plan original)

El README de este repo (secciones 4, 5, 7) decía XAdES-**BES**. Es incorrecto —
**el Anexo Técnico exige XAdES-EPES** (pág. 641): *"El formato XAdES de firma digital
avanzada adoptado por la DIAN... corresponde a la Directiva XAdES-EPES"*. Ya lo
corregí en el README (sección "Pendiente/Próximos pasos" al final de este
documento). La diferencia práctica: EPES añade una **política de firma declarada**
(un documento de políticas al que la firma hace referencia por URL + hash) — DSS
(la librería Java elegida) soporta XAdES-EPES de forma nativa, así que no cambia la
decisión de stack, solo la configuración exacta.

**Especificación exacta de la firma** (pág. 639-649):

| Parámetro | Valor |
|---|---|
| Formato | XMLDSig *enveloped* + XAdES-EPES (ETSI TS 101 903 v1.2.2/1.3.2/1.4.1) |
| Ubicación en el XML | `/Invoice/ext:UBLExtensions/ext:UBLExtension/ext:ExtensionContent/ds:Signature` (análogo para `CreditNote`, `DebitNote`, `ApplicationResponse`, `AttachedDocument`) |
| Canonicalización | `http://www.w3.org/TR/2001/REC-xml-c14n-20010315` (Canonical XML, sin comentarios) |
| Algoritmo de firma (`SignatureMethod`) | `rsa-sha256` (recomendado), también válidos `rsa-sha384`, `rsa-sha512`. **Nunca `rsa-sha1`** (caducado) |
| Algoritmo de digest (`DigestMethod` en cada `Reference`) | `http://www.w3.org/2001/04/xmlenc#sha256` |
| **URL de la política de firma** (`xades:SignaturePolicyId/xades:Identifier`) | `https://facturaelectronica.dian.gov.co/politicadefirma/v2/politicadefirmav2.pdf` |
| Hash de la política (`SigPolicyHash`) | SHA256 o SHA512 del PDF de la política de arriba |
| Descripción de la política | `"Política de firma para facturas electrónicas de la República de Colombia."` |
| `xades:SignerRole` | `"supplier"` (si firma el propio facturador) o `"third party"` (si firma un proveedor tecnológico en su nombre) |
| Certificado — uso de clave exigido | `Digital Signature` + `Non Repudiation` en el `X509v3 Key Usage` |
| Certificado — algoritmo de firma del certificado | `sha256WithRSAEncryption`, `sha384WithRSAEncryption` o `sha512WithRSAEncryption` (certificados emitidos después del 30-sep-2016; `sha1`/`sha224` solo para certificados más viejos, caso que no aplica a un certificado nuevo) |

Esto me da todos los parámetros concretos que hace falta pasarle a DSS cuando se
implemente el módulo de firma en la Fase 3 — no hay ambigüedad que resolver ahí,
solo falta escribir el código.

## 4. Webservices de la DIAN

**Protocolo** (pág. 320-324 del Anexo de Factura): SOAP 1.2, estilo Document/Literal,
sobre TLS 1.2 con autenticación mutua por certificado digital, usando el estándar
**WS-Security 1.0 OASIS con X.509 Certificate Token Profile 1.1** (esto es justo lo
que mencioné en el README como "cliente SOAP/XML maduro" — Apache CXF lo soporta de
forma nativa). Mismo protocolo para factura y para DEE POS — confirmado comparando
ambos Anexos.

**Los métodos — y aquí hay una diferencia real entre los dos documentos:**

| Método | Factura Electrónica | DEE POS (tiquete) | Para qué |
|---|---|---|---|
| `SendBillSync` | ✅ | ✅ | Individual, síncrono: enviar un documento y recibir la validación en la misma conexión |
| `SendBillAsync` | ✅ | ✅ *(mencionado en el texto, pero no en la lista introductoria — ver nota)* | Lote, asíncrono: ZIP con hasta 50 documentos firmados; devuelve `TrackId` |
| `SendTestSetAsync` | ✅ | ✅ | Solo ambiente de habilitación — envío de casos de prueba con un `testSetId` (UUID que la DIAN asigna por caso) |
| `SendEventUpdateStatus` | ✅ | ❌ no listado | Recepción de eventos (acuse de recibo, aceptación/rechazo del adquiriente) |
| `GetStatus` | ✅ | ✅ | Estado de un documento individual |
| `GetStatusZip` | ✅ | ✅ | Estado de un lote enviado con `SendBillAsync`, usando el `TrackId` |
| `GetNumberingRange` | ✅ | ✅ | Rango de numeración autorizado — de aquí sale la `ClTec` (factura) |
| `GetXmlByDocumentKey` | ✅ | ✅ | Descargar el XML de un documento ya emitido, por su CUFE/CUDE |

Nota: la lista introductoria del Anexo del DEE POS (pág. 1897) no menciona
`SendBillAsync` ni `SendEventUpdateStatus` explícitamente, pero el método
`SendBillAsync` sí aparece referenciado más adelante en el mismo documento (al
hablar de `GetStatusZip`). Puede ser una omisión del propio documento de la DIAN
más que una ausencia real del servicio — **a confirmar con el ambiente de
habilitación real antes de asumirlo**, no es algo que se pueda resolver solo
leyendo el PDF.

**Importante — no hay una URL de WSDL pública y fija.** El propio Anexo lo dice
(pág. 695 del de Factura): *"la URL del Web Service... estará expuesta en el
catálogo de participante (habilitación – producción) sobre la opción
Participantes"* — la DIAN asigna la URL real del webservice **por participante**,
visible solo después de registrarse, dentro del portal. Ya quedó reflejado en el
README como dato de configuración por tenant (sección 4.1).

**QR de verificación — ambos ambientes confirmados** (pág. 482 del Anexo DEE POS):
- Producción: `https://catalogo-vpfe.dian.gov.co/document/searchqr?documentkey=CUFE` (o `CUDE`)
- Habilitación: `https://catalogo-vpfe-hab.dian.gov.co/document/searchqr?documentkey=CUFE`

(Antes tenía esto marcado como "a confirmar" — ya está confirmado, el dominio de
producción efectivamente no lleva el sufijo `-hab`.)

## 5. Umbral de 5 UVT — confirmado, sin contradicción con lo ya documentado en Dorato

Verificado por una fuente secundaria (no encontré el número exacto "5 UVT" en el
cuerpo del Anexo Técnico en sí, parece estar en el texto de la Resolución base, no
en el Anexo): el tiquete POS/DEE POS simplificado solo es válido para ventas
**menores a 5 UVT** (~$261.870 COP en 2026). Por encima de eso, es obligatorio
emitir factura electrónica completa con NIT del comprador, sin importar si lo pide
o no. Esto coincide con lo que ya decía el README de `dorato-app`.

Nota aparte: encontré un umbral distinto de **100 UVT** en el Anexo de Factura
relacionado con requisitos de identificación del comprador en otro contexto — no
hay que confundirlo con el umbral de los 5 UVT, son reglas distintas.

## 6. Pendiente de esta fase

Lo que ya se cerró en esta segunda pasada:

- ~~Detalle específico del Anexo Técnico del DEE POS~~ **hecho** — revisado a
  fondo: alcance exacto (sección 8.2 del documento), CUDE con `Software-PIN` en vez
  de `ClTec` (verificado computacionalmente), métodos de webservice comparados
  contra los de Factura, política de firma confirmada igual.
- ~~Confirmar la URL real de producción del QR~~ **hecho**:
  `catalogo-vpfe.dian.gov.co` sin `-hab` (sección 4).

Lo que sigue abierto:

- **Catálogos completos** (unidad de medida, tipos de tributo, municipios/DIVIPOLA,
  tipos de documento): están en el "Suplemento D: Tablas de Contenidos de Elementos
  y de Atributos" (pág. 683) y "Suplemento E: Códigos de Productos" (pág. 696) del
  Anexo de Factura. Son tablas largas — mejor extraerlas directamente a las tablas
  de catálogo de la base de datos cuando se construya la Fase 2, en vez de copiarlas
  a este documento.
- **Guía de Habilitación** (Suplemento J, pág. 739): remite a
  https://www.dian.gov.co/impuestos/factura-electronica/lo-que-deberias-saber/Paginas/instructivos.aspx
  — no se visitó todavía esa página ni se extrajo el set de casos de prueba oficial
  que hay que pasar.
- **Confirmar si `SendBillAsync` y `SendEventUpdateStatus` existen de verdad para
  DEE POS** — el documento es inconsistente al respecto (sección 4), solo se
  resuelve con acceso real al ambiente de habilitación, no leyendo más el PDF.
- **El portal de "catálogo de participante"** donde se obtienen las URLs/PIN/clave
  técnica reales de cada tenant — no se visitó, requiere credenciales de
  habilitación que todavía no existen (prerrequisito de la sección 6 del README).
- **Revisar `dian-kit`** (el SDK en TypeScript) con más profundidad — puede ahorrar
  tiempo de implementación en la Fase 3 como referencia de cómo resolvieron la
  parte de firma/CUFE en código real, aunque el stack elegido para `emite` sea
  Java/Kotlin.

## 7. Correcciones que esto implicó en el README principal

Las tres ya quedaron aplicadas en el README:

- ~~Cambiar toda mención de XAdES-BES por XAdES-EPES~~ **hecho** (secciones 1, 4.3,
  5, 8).
- ~~Agregar los parámetros exactos de la firma (política, hash, algoritmo) a la
  sección 5~~ **hecho**.
- ~~Agregar que la URL del webservice de la DIAN es un dato por tenant, no una
  constante global, a la sección 4.1~~ **hecho**.
