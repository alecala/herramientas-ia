Titulo: Crear sistema que extraiga una tabla de un PDF y lo arroje en un CSV. 

Contexto: Los archivos PlanDePag*.pdf representan los planes de pago de un credito personal a 72 cuotas, extraidos en distintas fechas. El problema es que se requiere obtener la información de las tablas que contienen las cuotas pactadas. Cada cuota tiene el monto total, y la divisón de acuerdo a amortización a capital, intereses, estado y fecha de pago, entre otras. 

Requerimiento Técnico: Extraer en un archivo csv que tenga las siguientes columnas Filename,Row Number,Fecha Fin,Capital,Interes Corriente,Interes Mora,Seguros/FNG,Otros Conceptos,Total Exigible,Total Pagado,Estado,Pendiente por Pagar,Condonado,Saldo de Capital, la información dispuesta en las tablas de los archivos PlanDePag*.pdf; el separador de miles debe omitirse, el separador de decimales debe ser el ".", No incluir simbolo monetario. Si la fila de la tabla no se puede extraer completamente se debe omitir y alogarla en un archivo de fallos. El código debe ser en Golang 1.24 darwin/arm, y debe permitir que las Expresiones regulares sean faciles de modificar. 

Definition of Done: El archivo CSV debe al menos tener 72 línes más el header. Se debe identicar la causa del fallo. La covertura del código debe estar por arriba del 95%.
