# Arquitectura de la BBDD
La arquitectura constara de las tablas LGA_MODELOS que recopilará todos los modelos de formularios que se quieran tratar en esta gestión, LGA_PERMISOS recopilará los permisos a gestionar, LGA_VIA_ACCESO recopila las diferentes vías de acceso  a España disponibles y LGA_AUTORIZACIONES recopilará todas las autorizaciones y los datos necesarios para su configuración en la aplicación de tramitación de estas.

## La tabla LGA_MODELOS tendrá la siguiente estructura:

| NOMBRE COLUMNA |	PK |	TIPO          |	NULLABLE |	VALOR POR DEFECTO |	COMENTARIO |
|----------------|-----|------------------|----------|--------------------|------------|
| ID             |	Sí | VARCHAR2(4 BYTE) |	No       | |Identificador del formulario (EX00, EX01, EX02, …) |
| DES_MODELO | | VARCHAR2(300 CHAR) | No | | Descripción identificativa del modelo de formulario |

## La tabla LGA_PERMISOS tendrá la siguiente estructura:

| CAMPO | PK | TIPO | NULLABLE | VALOR POR DEFECTO | COMENTARIO |
|-------|----|------|----------|-------------------|------------|
| ID | Sí | VARCHAR2(3 BYTE) | No |  | Código de permiso correspondiente a la aplicación de extranjería |
| DES_PERMISO |  | VARCHAR2(300 CHAR) | No |  | Descripción identificativa del permiso |
| LUCRATIVO |  | VARCHAR2(1 BYTE) | No | ‘N’ | Lucrativo (S/N) |
| RESIDENCIA |  | VARCHAR2(1 BYTE) | Sí | ‘N’ | Es permiso de residencia |
| VIA_DEFECTO |  | VARCHAR2(3 BYTE) | Sí |  | Vía por defecto al abrir |
| MESES_VALIDEZ |  | NUMBER | Sí |  | Meses validez del Permiso |
| REGLAMENTO |  | VARCHAR2(11 BYTE) | Sí |  | Reglamento por defecto de la autorización |

## La tabla LGA_VIA_ACCESO tendrá la siguiente estructura:

|CAMPO			|PK	|TIPO				|NULLABLE	|VALOR POR DEFECTO	|COMENTARIO																		 |
|---------------|---|-------------------|-----------|-------------------|--------------------------------------------------------------------------------|
|ID				|Sí	|VARCHAR2(3 BYTE)	|No			|					|Identificador de la vía de acceso correspondiente a la aplicación de extranjería|
|DES_VIA_ACCESO	|	|VARCHAR2(300 CHAR)	|Sí			|					|Descripción identificativa de la vía de acceso (supuesto)                       |

## La tabla LGA_AUTORIZACIONES tendrá la siguiente estructura:

|CAMPO		|PK	|TIPO	            |NULLABLE	|VALOR POR DEFECTO	|COMENTARIO                                                   |
|-----------|---|-------------------|-----------|------------------|--------------------------------------------------------------|
|COD_MEYSS	|Sí	|VARCHAR2(6 BYTE)	|No		    |                  |  Código del MEYSS de identificación de la autorización       |
|ID_PERMISO	|	|VARCHAR2(3 BYTE)	|No		    |                  |  Código de permiso relacionado con la tabla LGA_PERMISOS     |
|ID_VIA		|	|VARCHAR2(3 BYTE)	|No		    |                  |  Vía de acceso correspondiente a la aplicación de extranjería|
|ID_MODELO  |   |VARCHAR2(4 BYTE)   |No         |                  | Identificador del modelo de formulario a rellenar para solicitar esta autorización|
|NUM_PLAZO  |   |NUMBER             |Sí         |                  | Número del plazo|
|TIPO_PLAZO |   |VARCHAR2(1 BYTE)   |Sí         |                  | Tipo de plazo: D - Días, M - Meses, A - Años|
|SILENCIO   |   |VARCHAR2(1 BYTE)   |Sí         |                  | Sentido del silencio: P - Positivo, N - Negativo|