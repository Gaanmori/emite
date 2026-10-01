# Emite

Motor de facturación electrónica para Colombia (Documento Equivalente Electrónico
POS + Factura Electrónica de Venta ante la DIAN), pensado desde el día uno para que
**cualquier negocio lo auto-aloje** con su propio certificado y su propia
habilitación — no como un servicio centralizado que factura en nombre de terceros.

Nace como el motor de facturación de [Dorato](https://github.com/Gaanmori/dorato-app)
(un POS de restaurante), pero el dominio de entrada es genérico ("una venta con sus
items, impuestos y totales"), no específico de un negocio — la idea es que cualquier
otra empresa pueda desplegar su propia instancia y adaptarla.

> **Estado: en planeación.** Todavía no hay código. Este README es el plan de trabajo
> completo: alcance legal, arquitectura, roadmap y prerrequisitos.

---

## 0. Por qué esto es legal (y bajo qué condición deja de serlo)

No es la idea en sí la que puede generar un problema regulatorio, sino *cómo se
opera* una vez que otras empresas la usan. Hay una línea concreta que no se puede
cruzar sin asumir un requisito mucho más grande:

| | **Facturador directo con software propio** | **Proveedor Tecnológico (PT)** |
|---|---|---|
| Quién transmite a la DIAN | Cada empresa, con **su propio** certificado y **su propia** habilitación | Un tercero, en nombre de **otras** empresas, desde infraestructura propia del PT |
| Requisitos | Software (puede ser de código abierto), certificado de firma digital propio, habilitación propia ante la DIAN | Todo lo anterior **más**: patrimonio líquido > 20.000 UVT (del orden de varios cientos de millones a ~1.000 millones de COP — verificar el valor UVT vigente, sube cada año), certificación ISO 27001, autorización específica de la DIAN como PT, infraestructura auditada |
| ¿Emite auto-alojado por cada empresa? | **Esto** — cada empresa sigue siendo facturador directo, solo que reutiliza el mismo código | No se activa |
| ¿Emite alojado centralmente, facturando por varias empresas a la vez? | No aplica | **Esto sí** — requeriría convertirse en PT autorizado |

**La clave:** publicar código (abierto o no) nunca es, por sí mismo, un acto regulado
por la DIAN. Lo que la DIAN regula es *quién transmite el documento y en nombre de
quién*. Si cada empresa que adopta Emite:

1. Tiene su propio RUT/NIT y su propia responsabilidad de IVA/régimen definida,
2. Compra su propio certificado de firma digital,
3. Pide su propia resolución de numeración en MUISCA,
4. Se habilita ante la DIAN con sus propias credenciales de prueba,
5. Despliega **su propia instancia**, con sus propios secretos, sin que nada pase
   por un servidor compartido,

...entonces sigue siendo, legalmente, "facturador directo con software propio" —
igual que dos empresas distintas usando ambas el mismo ERP de código abierto.

**Lo que sí convertiría esto en un problema regulatorio:** alojar una instancia
compartida que transmita documentos a la DIAN *en nombre de* varios NITs distintos
desde una misma infraestructura. Ese modelo (facturación como servicio centralizado,
tipo Alegra/Factus/Siigo) es el de un Proveedor Tecnológico, con los requisitos de
capital e ISO 27001 de la tabla de arriba.

**Esto ya se validó con un asesor** (octubre 2026): confirma que, mientras cada
empresa se auto-aloje con su propio certificado y su propia habilitación, no hay
problema. Aun así, antes de que una segunda empresa real lo use en producción,
vale la pena una segunda confirmación puntual sobre cualquier matiz nuevo de la DIAN
en el momento.

**Viabilidad técnica:** alcanzable. No es investigación de frontera — es implementar
un protocolo gubernamental bien documentado (XML UBL 2.1, firma XAdES-BES, cálculo
de CUFE/CUDE, llamadas a un webservice) que ya implementaron Alegra, Factus, Siigo,
MATIAS API y otros. Hay que mantenerlo cuando la DIAN cambie el Anexo Técnico, pero
nada ahí es imposible de construir con el tiempo adecuado.

**Realidad de mercado (para que la expectativa sea justa):** la mayoría de las
empresas pequeñas en Colombia prefieren pagar un PT en vez de auto-alojar su propio
motor, precisamente porque evita que ellas mismas gestionen certificado, habilitación
y mantenimiento. El público realista para Emite es más angosto: negocios con equipo
técnico propio (como Dorato), o desarrolladores que no quieren depender de un
proveedor pago. Si una agencia de software quiere usar Emite para varios clientes,
tiene que desplegar una instancia separada por cliente — una sola instancia
compartida para varios NITs vuelve a caer en terreno de Proveedor Tecnológico.

---

## 1. Qué es Emite

Un motor/API de facturación electrónica para Colombia, separado de cualquier negocio
concreto. El dominio de entrada es "una venta con sus items, impuestos y totales" —
no sabe nada de mesas, pizzas ni mitad-y-mitad; eso es responsabilidad de quien lo
integra (Dorato traduce su `pagos`/`items_pedido` a este contrato).

---

## 2. Licencia — decisión pendiente, con trade-offs

Afecta directamente el futuro del proyecto y hay que decidirlo *antes* de aceptar
código de terceros:

| Licencia | Qué permite | Riesgo/limitación |
|---|---|---|
| **MIT / Apache 2.0** (permisiva) | Cualquiera la usa, modifica y hasta la revende como servicio, sin obligación de devolver nada | Un competidor podría tomar el motor, alojarlo como servicio (ellos sí con status de PT) y competir sin pagar ni contribuir de vuelta |
| **AGPL-3.0** (copyleft fuerte) | Igual de abierta, pero si alguien la usa para dar un servicio (SaaS), está obligado a publicar sus modificaciones | Protege de que alguien la convierta en SaaS cerrado sin compartir cambios; puede espantar a empresas que no quieren tocar código AGPL por política interna |
| **BSL / Elastic License** (código visible, uso restringido) | Cualquiera la lee, la audita y la auto-aloja gratis; pero *ofrecerla como servicio hospedado a terceros* requiere licencia comercial | Modelo de Sentry, Elastic, MongoDB (antes). No es "open source" en el sentido estricto (OSI), aunque el código es público |
| **Dual-license (ej. AGPL + comercial)** | Gratis bajo AGPL para quien se auto-aloje y comparta cambios; licencia comercial de pago para incrustarla en software propietario | Modelo más común para monetizar un proyecto "open source" de verdad (GitLab, MinIO) |

**Recomendación:** empezar con BSL o dual-license (AGPL + comercial), no con
MIT/Apache puro, para no regalar la ventaja a un competidor desde el primer commit.
Pendiente de decisión — por eso **todavía no hay archivo `LICENSE`** en este repo
(sin uno, por defecto el código está bajo "todos los derechos reservados", que es la
postura más segura mientras se decide).

---

## 3. Modelo de despliegue: auto-alojado por tenant (no SaaS centralizado)

- **Un tenant = una empresa = sus propias credenciales, su propio certificado, su
  propia base de datos o esquema aislado.**
- El código es el mismo para todos, pero **nunca hay una sola base de datos ni un
  solo proceso transmitiendo documentos de varios NITs distintos**, salvo que el
  propio adoptante corra varias sedes bajo su mismo NIT (eso sí es válido).
- "Nosotros alojamos tu instancia por una tarifa" es viable después (sigue siendo
  "cada cliente con su instancia aislada"), pero antes de ofrecerlo hay que
  confirmarlo puntualmente con el asesor legal.

---

## 4. Alcance funcional del API (v1)

### 4.1 Configuración por tenant (una vez, al dar de alta una empresa)
- Datos del RUT: NIT, razón social, responsabilidad de IVA/régimen (ordinario vs
  SIMPLE — cambia qué documento se debe emitir).
- Certificado de firma digital (`.p12`/`.pfx` + contraseña), almacenado cifrado,
  nunca en texto plano ni junto al resto de la base de datos de negocio.
- Resoluciones de numeración (DEE POS y factura electrónica): prefijo, rango
  autorizado, vigencia.
- Ambiente activo: habilitación/pruebas vs producción.
- `testSetId` / credenciales que la DIAN entrega durante la habilitación de ese NIT.

### 4.2 Entrada: el contrato de "venta"
Una venta llega como un objeto neutral: items (nombre, cantidad, precio, tarifa de
IVA/impuesto aplicable), totales, método de pago, datos del comprador (opcional —
obligatorio solo si pide factura completa o si supera el umbral), fecha/hora,
identificador interno de la venta en el sistema de origen.

### 4.3 Lo que el motor hace con cada venta
1. Decide el tipo de documento: Documento Equivalente POS, o Factura Electrónica
   completa si el comprador pide NIT o si el total supera el umbral vigente (5 UVT a
   la fecha de la última revisión — verificar el valor actual).
2. Mapea los datos a los catálogos oficiales de la DIAN (unidad de medida, tipos de
   tributo, municipios/departamentos, tipo de documento, etc.).
3. Genera el XML UBL 2.1 correspondiente.
4. Lo firma digitalmente (XAdES-BES) con el certificado de ese tenant.
5. Calcula el CUFE (factura) o CUDE (DEE POS) con el algoritmo oficial.
6. Lo transmite al webservice de la DIAN (ambiente activo de ese tenant) y procesa
   la respuesta (aceptado, rechazado, con qué errores).
7. Si la DIAN está caída: lo deja en una cola de reintento ("pendiente de
   transmisión"), nunca se pierde ni bloquea la venta del lado del POS.
8. Genera la representación gráfica (PDF con código QR) una vez validado.
9. Expone el resultado: estado, CUFE/CUDE, XML firmado, PDF.
10. Notifica (webhook) cuando el estado cambia.

### 4.4 Endpoints (borrador, se refina al empezar a programar)
- `POST /tenants` — alta de una empresa nueva (RUT, certificado, resoluciones).
- `POST /tenants/{id}/ventas` — registra una venta y dispara el flujo de 4.3.
- `GET /tenants/{id}/ventas/{id}` — estado y documentos de una venta.
- `GET /tenants/{id}/ventas/{id}/pdf` — representación gráfica.
- `POST /tenants/{id}/ventas/{id}/reintentar` — fuerza un reintento en contingencia.
- Webhook saliente configurable por tenant.

---

## 5. Arquitectura técnica

- **No como Edge Function de Supabase.** La firma XAdES-BES, el manejo de
  certificados, las llamadas SOAP/XML a la DIAN y los catálogos grandes exceden lo
  cómodo para una función serverless ligera. Es un servicio backend aparte,
  desplegado en su propio sitio (VPS pequeño, Fly.io, Railway, Render — a decidir),
  con su propia base de datos.
- **Stack:** a decidir (sección 9). Candidatos: Node.js/TypeScript (buen ecosistema
  para XML/firma digital), Python, o Dart backend (`dart_frog`/`shelf`, reutiliza el
  lenguaje que ya domina el equipo, aunque el ecosistema de firma XAdES/XML-DSig en
  Dart es más limitado — revisar antes de elegir).
- **Secretos:** los certificados de cada tenant son el activo más sensible del
  proyecto. Necesitan un vault real (HashiCorp Vault, AWS/GCP Secrets Manager, o como
  mínimo cifrado por tenant con una clave que ni el operador del servicio pueda leer
  en texto plano sin un paso explícito) — nunca una columna de base de datos sin
  cifrar.
- **Aislamiento por tenant:** cada despliegue real debe tener su propia base de
  datos (o esquema) y su propio certificado — nunca mezclados (ver sección 3).

---

## 6. Prerrequisitos por cada empresa adoptante (incluido Dorato)

Se repite para **cada** empresa que use el motor — cada una necesita su propia
relación con la DIAN:

1. RUT actualizado con la actividad económica correcta y la responsabilidad de
   IVA/régimen definida.
2. Certificado de firma digital (persona jurídica) con una entidad certificadora
   acreditada (Certicámara, GSE, ANDES SCD, etc.) — tiene costo y vigencia limitada
   (1-2 años), hay que renovarlo.
3. Resoluciones de numeración en MUISCA (DEE POS y, si aplica, factura electrónica) —
   trámites separados.
4. Usuario y credenciales en el ambiente de habilitación/pruebas de la DIAN.
5. Pasar el set de casos de prueba oficial de la DIAN con el motor ya integrado,
   antes de poder facturar en producción.

---

## 7. Conocimiento técnico a reunir antes de programar (se puede empezar ya)

- Anexo Técnico oficial del DEE POS (Resolución 000165 de 2023, modificada por la
  000202 de 2025, compilada en la Resolución Única 000227 de 2025).
- Anexo Técnico de Factura Electrónica de Venta (UBL 2.1 Colombia).
- Catálogos oficiales completos: unidad de medida, tipos de tributo, municipios y
  departamentos, tipos de documento, etc.
- Algoritmo exacto de cálculo de CUFE/CUDE (orden de concatenación de campos,
  función de hash usada).
- Contratos de los webservices de la DIAN (habilitación y producción): formato de
  petición/respuesta, manejo de errores, límites de tamaño/tasa.
- El set de casos de prueba que la DIAN exige pasar en habilitación.

---

## 8. Roadmap por fases

**Fase 0 — Legal y estrategia**
~~Confirmar con abogado/contador que el modelo auto-alojado evita el estatus de
Proveedor Tecnológico~~ **hecho (octubre 2026)**. ~~Decidir nombre del proyecto~~
**hecho: Emite**. Pendiente: licencia (sección 2).

**Fase 1 — Investigación y especificación** *(sin prerrequisitos, se puede arrancar ya)*
Todo lo de la sección 7: estudiar los dos Anexos Técnicos, documentar catálogos,
algoritmo CUFE/CUDE, contratos de los webservices, casos de prueba de habilitación.

**Fase 2 — Dominio central y modelo multi-tenant** *(código, sin credenciales reales
todavía)*
Diseñar el modelo de tenant (4.1), el contrato de entrada "venta" (4.2) desacoplado
de cualquier negocio concreto, las tablas de catálogos, y el generador de XML UBL 2.1
(DEE POS primero) contra datos de prueba.

**Fase 3 — Criptografía y transmisión a la DIAN**
Firma XAdES-BES, cálculo de CUFE/CUDE, cliente del webservice (ambiente de
habilitación), cola de contingencia/reintento.

**Fase 4 — Alta de tenants**
Almacenamiento seguro de certificados (vault), flujo de alta de una empresa nueva,
documentación para que una empresa externa configure su propia instancia.

**Fase 5 — Certificación** *(se repite por cada empresa adoptante, incluida Dorato)*
Correr el set oficial de pruebas de habilitación de la DIAN con las credenciales de
esa empresa, corregir lo que falle, pasar a producción.

**Fase 6 — Integración de Dorato (primer consumidor real)**
Conectar el flujo de `cobrar_mesa`/`pagos` de Dorato a Emite en vez del mock actual
de `recibo_pdf.dart`. Reemplazar el PDF simulado por la representación gráfica real
con CUFE/CUDE y QR validados.

**Fase 7 — Publicación como proyecto reutilizable**
Verificar que el núcleo quede libre de cualquier cosa específica de Dorato.
Documentación de puesta en marcha para terceros. Publicar con la licencia elegida.

**Fase 8 — Mantenimiento continuo** *(para siempre — el motivo de hacerlo
compartido: el costo de mantenerlo al día con la DIAN se reparte entre todos los que
lo usan)*
Seguimiento de cambios al Anexo Técnico, renovación de certificados por tenant,
parches de seguridad, revisión de aportes de la comunidad.

---

## 9. Decisiones pendientes

1. ~~Nombre del proyecto~~ **hecho: Emite.**
2. ~~Validación legal inicial~~ **hecho (octubre 2026).**
3. **Licencia** (sección 2): ¿MIT/Apache, AGPL, BSL/dual-license?
4. **Stack técnico**: ¿Node/TypeScript, Python, o Dart backend?
5. **Prioridad relativa**: ¿esto pasa a ser lo siguiente a trabajar, o sigue en cola
   detrás de lo pendiente de arquitectura de Dorato?

---

## 10. Cómo encaja Dorato

Dorato sigue siendo, legalmente, un "facturador directo con software propio" — nada
cambia para Dorato en términos de obligaciones ante la DIAN (sección 6 aplica igual).
Lo único que cambia es que el motor que resuelve esa obligación no vive dentro de
`dorato-app`, sino aquí, y Dorato lo consume como un cliente más (el primero). Cuando
llegue la Fase 6, el código de Dorato que hoy simula el ticket (`recibo_pdf.dart`,
`EmpresaConfig`) se reemplaza por una llamada real a esta API.
