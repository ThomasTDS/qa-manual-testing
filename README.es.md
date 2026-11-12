# Pruebas Manuales y Exploratorias

![Licencia](https://img.shields.io/badge/licencia-MIT-blue.svg)

*[Leia em português](README.md) · [Read in English](README.en.md)*

Portafolio de artefactos de pruebas manuales y exploratorias: diseño de
casos de prueba con trazabilidad de reglas de negocio, informes de sesiones
de pruebas exploratorias (Session-Based Test Management) y reportes de
defectos bien documentados.

Este repositorio complementa el portafolio de automatización de pruebas del
mismo autor y está en construcción — el contenido se agrega de forma
incremental.

## Destacados

Tres hallazgos que resumen el tipo de razonamiento que este portafolio intenta demostrar — el proceso detrás de cada uno importa más que la lista en sí:

- **[Una cuenta que queda atrás sin aviso (BugBank)](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md).** BugBank no tiene un botón de "olvidé mi contraseña". El camino más obvio que intentaría un usuario real — registrarse de nuevo con el mismo correo electrónico — se acepta sin ningún aviso de duplicidad y crea una segunda cuenta completamente desconectada de la primera. El hallazgo no surgió de probar un escenario de bug evidente; surgió de simular el comportamiento de alguien que solo quiere volver a acceder a su propia cuenta.

- **[Un préstamo aprobado que deja la cuenta en números rojos (ParaBank)](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md).** La regla de aprobación de ParaBank suma el saldo de todas las cuentas del cliente para decidir si aprueba un préstamo, pero debita el pago inicial de una sola cuenta específica, sin verificar si esa cuenta por sí sola tiene fondos suficientes. Una solicitud aprobada correctamente por la regla puede dejar una cuenta con saldo negativo, sin aviso — y el efecto se acumula en solicitudes futuras.

- **Una hipótesis probada y descartada, no solo confirmada.** No todo riesgo señalado en un diseño de pruebas se convierte en defecto: el monto de préstamo negativo, marcado como posible inconsistencia en ParaBank, se ejecutó de verdad y la aplicación lo manejó correctamente. Documentar esto importa tanto como documentar un bug — demuestra que la conclusión vino de una ejecución real, no de una suposición.

Cómo se obtuvo cada hallazgo (lectura del código fuente, especificación oficial de la aplicación, ejecución real) se detalla en [`metodologia.md`](metodologia.md) *(solo en portugués)*.

## Estructura

| Carpeta | Contenido |
| --- | --- |
| [`casos-de-teste/`](casos-de-teste/) | Diseño de casos de prueba con trazabilidad de reglas de negocio, documentando la técnica utilizada (tabla de decisión, partición de equivalencia, análisis de valores límite) para cada funcionalidad analizada |
| [`sessoes-exploratorias/`](sessoes-exploratorias/) | Informes de sesiones de pruebas exploratorias (Session-Based Test Management): charter, cobertura por área, registro cronológico de pruebas y defectos confirmados |
| [`relatorios-de-bugs/`](relatorios-de-bugs/) | Reportes de defectos reales encontrados durante las sesiones de prueba, con pasos de reproducción, evidencia, análisis técnico de la causa y justificación de severidad/prioridad |

> Los artefactos enlazados a continuación están escritos en portugués, el idioma de trabajo del proyecto. Este README es una guía en español del contenido del repositorio.

## Artefactos

### ParaBank

- [Solicitar Préstamo](casos-de-teste/parabank-solicitar-emprestimo.md):
  regla de negocio extraída directamente del código fuente oficial de la
  aplicación, tabla de decisión y dieciséis casos de prueba trazados hasta
  la regla que cada uno valida.
- [Sesión exploratoria — Solicitar Préstamo](sessoes-exploratorias/parabank-solicitar-emprestimo-22-09-2026.md):
  ejecución real contra la aplicación, con dos defectos confirmados (error
  interno con un monto de préstamo igual a cero y débito del pago inicial
  sin verificar el saldo individual de la cuenta de origen) y una
  corrección aplicada al artefacto de diseño de casos de prueba a partir de
  lo observado.
- [BUG-EMP-001 — Un monto de préstamo igual a cero provoca un error interno del servidor](relatorios-de-bugs/bug-emp-001-valor-zero-erro-500.md):
  severidad alta, prioridad media.
- [BUG-EMP-002 — La aprobación del préstamo permite saldo negativo en la cuenta de origen](relatorios-de-bugs/bug-emp-002-debito-sem-validar-conta-origem.md):
  severidad crítica, prioridad alta, con efecto cascada confirmado sobre
  solicitudes de préstamo futuras.

### BugBank

- [Transferencia](casos-de-teste/bugbank-transferencia.md):
  regla de negocio extraída de la especificación oficial publicada por la
  propia aplicación, tabla de decisión con tres condiciones combinadas y
  catorce casos de prueba, incluidos escenarios que la especificación no
  cubre explícitamente.
- [Sesión exploratoria — Registro, Inicio de sesión y Transferencia](sessoes-exploratorias/bugbank-transferencia-22-09-2026.md):
  ejecución manual a través de la interfaz, con dos defectos confirmados
  (cuenta duplicada al volver a registrarse con el mismo correo, sin flujo
  de recuperación de contraseña; y validación inconsistente del formulario
  de transferencia, incluido un mensaje técnico de una biblioteca expuesto
  al usuario) y la confirmación de una regla funcionando correctamente.
- [BUG-CAD-001 — Volver a registrarse con un correo existente crea una cuenta paralela](relatorios-de-bugs/bug-cad-001-recadastro-sem-verificar-email-duplicado.md):
  severidad alta, prioridad alta.
- [BUG-TRF-001 — La validación del formulario de transferencia es inconsistente](relatorios-de-bugs/bug-trf-001-validacao-formulario-inconsistente.md):
  severidad media, prioridad media.

## Autor

Desarrollado por [Thomas Teixeira](https://github.com/ThomasTDS) como
proyecto de estudio y portafolio en pruebas de software.
