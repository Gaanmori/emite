# Emite

Motor de facturación electrónica para Colombia (Documento Equivalente Electrónico
POS + Factura Electrónica de Venta ante la DIAN), pensado desde el día uno para que
**cualquier negocio lo auto-aloje** con su propio certificado y su propia
habilitación — no como un servicio centralizado que factura en nombre de terceros.

Nace como el motor de facturación de [Dorato](https://github.com/Gaanmori/dorato-app)
(un POS de restaurante), pero el dominio de entrada es genérico ("una venta con sus
items, impuestos y totales"), no específico de un negocio — la idea es que cualquier
otra empresa pueda desplegar su propia instancia y adaptarla.

**Dorato es la prueba piloto, no el objetivo final.** La meta a mediano plazo es
poder cobrar por el uso de Emite a otras empresas (ver sección 2b, "Modelo de
negocio"). Eso condiciona varias decisiones de abajo (licencia, modelo de
despliegue) que de otra forma serían más simples.

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

**⚠️ Pregunta nueva y pendiente para el mismo asesor, específica por el modelo de
negocio (octubre 2026):** la validación que ya se hizo cubre el caso en que *cada
empresa despliega y opera su propia instancia*. Pero si el plan es cobrar por
operar/alojar nosotros mismos la instancia de cada cliente (aunque esté técnicamente
aislada — su propia base de datos, su propio certificado, su propio NIT), eso es un
escenario distinto que **no quedó cubierto por esa primera confirmación**. La
diferencia entre "vender software que tú instalas" y "operar la infraestructura de
facturación de un tercero por una tarifa" es exactamente la frontera que separa a un
facturador directo de un Proveedor Tecnológico — y "quién tiene las manos en el
servidor" puede pesar tanto como "a nombre de quién se transmite". Antes de ofrecer
alojamiento gestionado como producto de pago (sección 2b), hay que llevarle al
asesor esta pregunta puntual: *"si nosotros alojamos y operamos la instancia aislada
de cada cliente, cobrándoles una tarifa, pero cada cliente mantiene su propio
certificado y su propia habilitación ante la DIAN, ¿seguimos siendo 'facturador
directo con software propio' por cada cliente, o eso ya nos acerca al estatus de
Proveedor Tecnológico?"*

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

## 2. Licencia: BSL 1.1 (decidido, octubre 2026)

Afecta directamente el futuro del proyecto y hay que decidirlo *antes* de aceptar
código de terceros:

| Licencia | Qué permite | Riesgo/limitación |
|---|---|---|
| **MIT / Apache 2.0** (permisiva) | Cualquiera la usa, modifica y hasta la revende como servicio, sin obligación de devolver nada | Un competidor podría tomar el motor, alojarlo como servicio (ellos sí con status de PT) y competir sin pagar ni contribuir de vuelta |
| **AGPL-3.0** (copyleft fuerte) | Igual de abierta, pero si alguien la usa para dar un servicio (SaaS), está obligado a publicar sus modificaciones | Protege de que alguien la convierta en SaaS cerrado sin compartir cambios; puede espantar a empresas que no quieren tocar código AGPL por política interna |
| **BSL / Elastic License** (código visible, uso restringido) | Cualquiera la lee, la audita y la auto-aloja gratis; pero *ofrecerla como servicio hospedado a terceros* requiere licencia comercial | Modelo de Sentry, Elastic, MongoDB (antes). No es "open source" en el sentido estricto (OSI), aunque el código es público |
| **Dual-license (ej. AGPL + comercial)** | Gratis bajo AGPL para quien se auto-aloje y comparta cambios; licencia comercial de pago para incrustarla en software propietario | Modelo más común para monetizar un proyecto "open source" de verdad (GitLab, MinIO) |

**Decidido: BSL 1.1.** Encaja con el modelo de negocio basado en alojar instancias
(sección 2b) en vez de vender módulos cerrados: evita que alguien tome el motor y
lance su propio "Emite hosteado" compitiendo contigo con tu propio código, sin
cerrarle la puerta a quien solo quiere auto-alojarlo gratis para sí mismo.

El texto completo está en [`LICENSE`](LICENSE) (plantilla oficial de BSL 1.1, sin
modificar más que los parámetros). Lo que dice en criollo:

- **Gratis, incluso en producción:** auto-alojar Emite para emitir tus propios
  documentos fiscales, bajo tu propio NIT (o el de empresas bajo tu mismo control).
  Esto cubre a Dorato y a cualquier empresa que se auto-aloje su propia instancia
  (sección 3) sin pagarte nada.
- **Requiere licencia comercial tuya:** ofrecer la funcionalidad de Emite — entera,
  parcial, o modificada — como servicio alojado/gestionado a un tercero. Esto es
  exactamente la opción 3 del modelo de negocio (sección 2b): si Emite mismo la
  ofrece bajo este esquema, lo hace con tu propia licencia comercial; si alguien más
  quisiera copiar el motor y montar su propio negocio de alojamiento, necesitaría
  negociar contigo primero.
- **Fecha de cambio: 1 de octubre de 2030.** A partir de esa fecha (para el código
  publicado hasta ese momento), la licencia pasa automáticamente a Apache 2.0 —
  completamente libre, sin restricciones. Es la característica que distingue a BSL
  de una licencia propietaria pura: no es "cerrado para siempre", tiene una fecha de
  vencimiento incorporada hacia código abierto de verdad.
- **Licenciante:** quedó a tu nombre (Gabriel Montoya) porque no existe todavía una
  empresa constituida para el proyecto. Si en algún momento creas una, es cuestión
  de actualizar ese campo.

**Importante:** usé la plantilla oficial de BSL 1.1 tal cual la publica MariaDB
(su texto es reutilizable por diseño), pero **la redacción de la cláusula de uso
adicional** (qué cuenta como "ofrecer como servicio a terceros") la escribí yo para
este caso concreto — es la parte que de verdad vale la pena que revise tu abogado
antes de que alguien dependa de ella comercialmente, junto con el resto de la
validación legal pendiente (sección 0).

---

## 2b. Modelo de negocio: cómo se monetiza

Confirmado (octubre 2026): la intención es cobrar por el uso de Emite a futuro.
Dorato es la prueba piloto para validar que el motor funciona de punta a punta antes
de ofrecerlo a nadie más.

Dentro de la restricción legal de la sección 0 (no se puede operar una sola
infraestructura compartida facturando por varios NITs sin ser Proveedor
Tecnológico), hay tres formas de cobrar que sí encajan, de la más simple/segura a la
que necesita más confirmación legal:

1. **Soporte y mantenimiento por suscripción** (la más limpia legalmente): el
   código se auto-aloja la propia empresa cliente, nunca toca infraestructura de
   Emite, pero paga una tarifa recurrente por soporte, ayuda de puesta en marcha, y
   sobre todo por **mantenerse al día con los cambios del Anexo Técnico de la DIAN**
   (justo el trabajo de la Fase 8 del roadmap). Esto es vender un servicio
   profesional, no operar su facturación — no hay zona gris aquí.
2. **Licencia comercial** (vía BSL o dual-license, sección 2): empresas que quieren
   auto-alojar el motor pero sin cumplir las condiciones de la licencia abierta (ej.
   no quieren compartir sus modificaciones, o quieren incrustarlo en software
   propietario) pagan por una licencia sin esas restricciones. Tampoco hay zona
   gris: siguen auto-alojando, solo cambian los términos bajo los que usan el código.
3. **Alojamiento gestionado, instancia por cliente** (la que probablemente más se
   parece a "cobrar por el uso del API" en la intención original): Emite despliega y
   opera la instancia de cada cliente, aislada (su propia base de datos, su propio
   certificado, su propio NIT), y cobra una tarifa mensual o por documento emitido.
   Es el modelo más parecido a un negocio SaaS tradicional.

**Decisión (octubre 2026): se construyen las tres, en paralelo.** La pregunta legal
puntual de la opción 3 (marcada con ⚠️ en la sección 0) queda a cargo de Gabriel con
su abogado — no bloquea el trabajo de ingeniería (sí se puede diseñar y construir la
capacidad de alojamiento gestionado mientras se resuelve), pero **sí bloquea
ofrecérselo como producto de pago a un cliente real** hasta tener la confirmación.
Construir ≠ vender: se puede tener la infraestructura lista y validada con la propia
instancia piloto de Dorato sin que eso cuente como "ya se está cobrando por
alojamiento a terceros".

No hay que elegir solo una de las tres — son compatibles entre sí (de hecho 1 y 3
normalmente van juntos: quien paga por alojamiento también está pagando, de facto,
el mantenimiento).

---

## 3. Modelo de despliegue: auto-alojado por tenant (no SaaS centralizado)

- **Un tenant = una empresa = sus propias credenciales, su propio certificado, su
  propia base de datos o esquema aislado.**
- El código es el mismo para todos, pero **nunca hay una sola base de datos ni un
  solo proceso transmitiendo documentos de varios NITs distintos**, salvo que el
  propio adoptante corra varias sedes bajo su mismo NIT (eso sí es válido).
- El alojamiento gestionado por tarifa (ver modelo de negocio, sección 2b, opción 3,
  **confirmado para construir**) sigue este mismo principio — "cada cliente con su
  instancia aislada", nunca una base de datos compartida entre NITs distintos —
  aunque sea Emite quien la despliegue y la opere. Lanzarlo como producto de pago
  a un cliente real espera la confirmación legal puntual marcada con ⚠️ en la
  sección 0 (a cargo de Gabriel con su abogado).

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
- **Stack: Java/Kotlin** (decidido, octubre 2026). Se eligió sobre Node/TS, Python y
  Dart específicamente por la pieza más delicada de todo el proyecto — la firma
  XAdES-BES —, donde Java tiene la librería más madura y probada en producción:
  [DSS (Digital Signature Services)](https://github.com/esig/dss) de la Comisión
  Europea, el estándar de facto para XAdES, usado por varias integraciones de
  facturación electrónica en LatAm. También aporta un cliente SOAP/XML maduro
  (Apache CXF/JAX-WS) para los webservices de la DIAN, y manejo nativo de
  certificados `.p12`/`.pfx` vía `KeyStore`. El costo asumido conscientemente: es un
  ecosistema nuevo para el equipo (hoy Flutter/Dart + TS en las Edge Functions de
  Dorato) — pero dado que esto se certifica ante un gobierno y los errores tienen
  consecuencias reales (sección 0), se prioriza la robustez de la librería de firma
  sobre la familiaridad del lenguaje. Kotlin es aceptable sobre la JVM si se prefiere
  una sintaxis más moderna que Java puro — misma librería (DSS), mismo ecosistema.
  Framework de API sugerido: Spring Boot o Ktor (más liviano, más idiomático en
  Kotlin) — a definir al empezar la Fase 2.
- **Cómo se conecta con Dorato (o cualquier otro consumidor): HTTP/REST, nada más.**
  El lenguaje del motor es invisible para quien lo consume — Dorato sigue siendo
  Flutter/Dart y nunca necesita saber que Emite está en Java, de la misma forma que
  hoy no le importa en qué lenguaje está escrita la API de Supabase que ya consume.
  El detalle de cómo se conecta concretamente está en la sección 10.
- **Secretos:** los certificados de cada tenant son el activo más sensible del
  proyecto. Necesitan un vault real (HashiCorp Vault, AWS/GCP Secrets Manager, o como
  mínimo cifrado por tenant con una clave que ni el operador del servicio pueda leer
  en texto plano sin un paso explícito) — nunca una columna de base de datos sin
  cifrar.
- **Aislamiento por tenant:** cada despliegue real debe tener su propia base de
  datos (o esquema) y su propio certificado — nunca mezclados (ver sección 3).
- **Aprovisionamiento repetible, por el alojamiento gestionado (sección 2b, opción
  3):** si Emite va a desplegar y operar la instancia de cada cliente, hace falta
  automatizar ese despliegue desde el principio (contenedor/imagen por tenant,
  variables de entorno o secretos inyectados por instancia, una base de datos nueva
  por cliente) en vez de hacerlo a mano cliente por cliente — si no, no escala más
  allá de dos o tres clientes. Conviene diseñarlo así desde la Fase 2, aunque el
  primer cliente real (Dorato) se despliegue manualmente al principio.
- **Métricas/facturación por tenant:** para cobrar por documento emitido o por
  suscripción (opción 3), cada instancia necesita reportar su propio consumo (cuántos
  documentos emitió, en qué período) a algún punto central de facturación — aunque
  los datos fiscales de cada tenant sigan completamente aislados entre sí, ese conteo
  de uso si puede centralizarse sin pisar la frontera legal de la sección 0 (es
  telemetría de uso, no transmisión de documentos a la DIAN en nombre de nadie).

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
**hecho: Emite**. ~~Decidir stack técnico~~ **hecho: Java/Kotlin**. Pendiente:
licencia (sección 2).

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

**Fase 4b — Infraestructura de alojamiento gestionado** *(nueva, octubre 2026 —
confirmada para construir, ver sección 2b opción 3)*
Automatizar el aprovisionamiento de una instancia aislada nueva por cliente
(contenedor/imagen + base de datos propia + secretos propios), y el reporte de
consumo por tenant hacia un punto central de facturación (sección 5). Se puede
construir y probar con la propia instancia de Dorato sin que eso cuente como
"vendiendo alojamiento a terceros" — **lanzarlo como producto de pago a un cliente
real espera la confirmación legal puntual de la sección 0.**

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
2. ~~Validación legal inicial (modelo auto-alojado por cada empresa)~~ **hecho
   (octubre 2026).**
3. ~~¿Se monetiza?~~ **hecho: sí, a futuro, con las 3 vías de la sección 2b.**
   Dorato es la prueba piloto.
4. ~~¿Se construye también el alojamiento gestionado (opción 3)?~~ **hecho: sí, en
   paralelo a las otras dos.**
5. **⚠️ En trámite — a cargo de Gabriel con su abogado:** validación legal específica
   del alojamiento gestionado (sección 0) — ¿operar la instancia aislada de cada
   cliente, cobrando una tarifa, sigue siendo "facturador directo" por cliente, o
   acerca a Proveedor Tecnológico? No bloquea construir la infraestructura (Fase 4b),
   solo bloquea venderla a un cliente real.
6. ~~Licencia~~ **hecho: BSL 1.1** (sección 2, archivo [`LICENSE`](LICENSE)).
7. ~~Stack técnico~~ **hecho: Java/Kotlin** (sección 5).
8. **Prioridad relativa**: ¿esto pasa a ser lo siguiente a trabajar, o sigue en cola
   detrás de lo pendiente de arquitectura de Dorato?

---

## 10. Cómo encaja Dorato

Dorato sigue siendo, legalmente, un "facturador directo con software propio" — nada
cambia para Dorato en términos de obligaciones ante la DIAN (sección 6 aplica igual).
Lo único que cambia es que el motor que resuelve esa obligación no vive dentro de
`dorato-app`, sino aquí, y Dorato lo consume como un cliente más (el primero). Cuando
llegue la Fase 6, el código de Dorato que hoy simula el ticket (`recibo_pdf.dart`,
`EmpresaConfig`) se reemplaza por una llamada real a esta API.

### 10.1 Cómo se conectan en la práctica (Flutter/Dart + Supabase → Emite en Java)

No hay ningún acoplamiento de lenguaje: Dorato nunca llama directamente a clases
Java, solo hace peticiones HTTP a un API REST, exactamente como ya hace hoy contra
Supabase. El flujo pensado (se afina en la Fase 6, pero esta es la forma):

1. **La venta en Dorato no espera a la DIAN.** `cobrar_mesa()` (ya construida, ver
   `supabase/README.md` de `dorato-app`) sigue resolviéndose igual de rápido que
   hoy — libera la mesa y registra el pago al instante. El documento fiscal se
   genera *después*, como un paso aparte, para que una DIAN lenta o caída nunca
   bloquee a un cajero cobrando en el restaurante (esto ya estaba previsto en la
   sección 4.3, punto 7).
2. **El disparo no depende de que la app de Flutter siga abierta.** En vez de que el
   propio cliente Flutter llame a Emite después de cobrar (si se cierra la app a
   medio camino, se perdería el intento), lo dispara el *servidor*: un **Database
   Webhook de Supabase** (o un trigger que llama a `pg_net`) en el `INSERT` de la
   tabla `pagos` invoca una Edge Function nueva de Dorato, p. ej.
   `emitir-documento-fiscal`.
3. **Esa Edge Function arma el payload** con el contrato de "venta" de la sección 4.2
   (traduce `pagos`/`items_pedido`/`productos` al formato neutral que espera Emite)
   y hace un `POST /tenants/{tenantDorato}/ventas` contra la API de Emite.
4. **Las credenciales de Dorato ante Emite** (su API key de tenant) viven como
   secreto de Supabase (mismo patrón que ya usa la `service_role key` en las Edge
   Functions actuales) — nunca llegan al cliente Flutter.
5. **Emite responde async.** Cuando la DIAN valida (o rechaza) el documento, Emite
   llama de vuelta a un webhook de Dorato (otra Edge Function pequeña,
   `webhook-emite`) con el resultado: CUFE/CUDE, XML firmado, PDF, o el motivo de
   rechazo si falló.
6. **Dorato guarda el resultado** (una tabla nueva, algo como
   `documentos_fiscales`, o una columna sobre `pagos`) y la pantalla de ticket
   (`ReciboPreviewScreen`, hoy con datos simulados) pasa a mostrar el documento real
   una vez llega ese webhook — con un estado intermedio ("procesando con la DIAN")
   mientras tanto.

Este mismo patrón (API key de tenant guardada como secreto server-side + webhook de
vuelta) es el que usaría *cualquier* otra empresa que adopte Emite, sin importar en
qué esté escrito su propio sistema — Dorato no tiene ningún trato especial aquí más
que ser la primera en probarlo.
