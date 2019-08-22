# **Grupo Inteca, Cartago**
Proyecto de Facturación Electrónica, Agosto 2018

## Desarrolladores
- Jairo Martínez
- Jeffrey Camareno

## Administrador del proyecto
- Jeffrey Camareno


# Tabla de contenidos
1. [Creación de directorios](#uno)
2. [Scripts  de permisos y seguridad](#dos)
3. [Importar el .dll del CLR](#tres)
4. [Crear el Store Procedure (SP) a través del CLR importado](#cuatro)
5. [Crear un linked server](#cinco)   
6. [Creación de tablas de parámetros](#seis)
7. [Creación de la vista de identificaciones](#siete)
8. [Funciones](#ocho)
9. [Crear tabla de logs](#nueve)
10. [Crear Store Procedure (SP) Insert_EInvoice 4.2 (NO VIGENTE)](#diez) 
11. [Crear Store Procedure (SP) Insert_EInvoice 4.3 (VIGENTE)](#once)
12. [Crear el Store Procedure (SP) Check_Insert_EInvoicelog](#doce)
13. [Crear Job EInvoiceFMCM](#trece)

## Manual de Instalación del Sistema de Facturación Electrónica en los POS-HOMEX
Para la implementación de este sistema fue necesario intregrar diferentes tecnologías, tales como:
- web services
- CLR con visual studio para crear store procedures para SQL Server 2008 R2 desde C# para compilarlo como un .dll
- SQL Server
- X++, Microsoft Dynamics AX 2012

La solución para este proyecto se hizo en su mayoría por medio de la base de datos de cada POS, por lo cual, la instalación se tiene que hacer en la base de datos del servidor de cada POS-HOMEX.

## Paso a paso
### 1. Creación de directorios <a name="uno"></a>
Ingresar al servidor de cada POS-HOMEX y crear las siguientes carpetas en la raíz del disco duro principal del servidor (_usualmente en la unidad C_)

    |-- FacturaElectronica
        |-- Homex
            |-- DocumentosEnviados
            |-- DocumentosInsertados
            |-- Install

### 2. Scripts  de permisos y seguridad <a name="dos"></a>
Ingresar a SQL Server Management Studio en cada POS y ejecutar los siguientes scripts:
**Nota 1:** _Ejecutar cada script en la base de datos transaccional del POS. Ej: RetailStore, RetailChanelDB, etc._
**Nota 2:** _Ejecutar cada línea por separado_

```SQL
SET NOCOUNT ON;

sp_configure 'show advanced options', 1;  
GO

RECONFIGURE; 
GO

sp_configure 'clr enabled', 1; 
GO

RECONFIGURE;
GO

ALTER DATABASE [DataBaseName] SET TRUSTWORTHY ON;
```

### 3. Importar el .dll del CLR <a name="tres"></a>
Dicho _.dll_ lo que contiene es la programación C# del método que se encarga de Insertar Documentos Electrónicos (enviar los XMLs de las facturas) al Ministerio de Hacienda de Costa Rica. Dicho método se conecta al _Web Service_ del proveedor.

El archivo _.dll_ lo puede descargar aquí https://github.com/Jef7/EInvoice-Homex/raw/master/SQLCLREINVOICE.dll

Guardar el archivo dentro de la carpeta **Install**

**Nota 1:** _Antes de importar el .dll, debe asegurarse que usted es el owner de la base de datos de Retail.
En caso de que no seas el owner, ejecutar el siguiente script:_
```SQL
EXEC dbo.sp_changedbowner @loginame = N'inteca\YourUserName'
```

**NOTA 2:** _En caso de que se muestre el siguiente error: "The proposed new database owner is already a user or aliased in the database". Ejecutar los siguientes comandos con el usuario que desea ser el nuevo owner:_

```SQL
GO
SP_DROPUSER 'inteca\YourUserName'

GO
SP_CHANGEDBOWNER 'inteca\YourUserName'
```

Siendo el _owner_ de la base de datos, importar el _.dll_ en la siguiente carpeta de SQL Databases/_DatabaseName_/Programmability/Assemblies

Path to assembly: (buscar el .dll descargado en la carpeta Install)

**Nota 3:** _Asegurarse de importar el .dll con **set permmission en External Access**_

### 4. Crear el Store Procedure (SP) a través del CLR importado <a name="cuatro"></a>
**Nota 1:** _Antes de crear el SP, debe correr el siguiente script para darle los permisos a la base de datos:_
```SQL
ALTER ASSEMBLY [SQLCLREINVOICE] WITH PERMISSION_SET = UNSAFE;
```

**Nota 2:** _En caso de que el servidor sea de 32 bits y se muestre el siguiente error: "Memory pressure errors Failed to initialize the Common Language Runtime (CLR) v2.0.50727 due to memory pressure", seguir los pasos que se indican en los siguientes enlaces:_ 
- [Memory pressure errors](https://support.microsoft.com/en-us/help/969962/various-memory-errors-are-logged-to-sql-server-error-log-when-using-sq) 
- [Solución](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2005/ms345416(v=sql.90)) 

Luego de dar los permisos a la base de datos, ejecutar los siguientes scripts para crear el SP:
```SQL
CREATE PROCEDURE [dbo].[InsertarDocumentosEInvoice](
@_pvcDocumentosXML Nvarchar(max), 
@_pvcCorreoUsuario Nvarchar(30), 
@_pvcClaveUsuario Nvarchar(30), 
@_invoiceId Nvarchar(50), 
@_sucursal Nvarchar(3), 
@_terminal Nvarchar(73)
)

WITH EXECUTE AS CALLER
AS EXTERNAL NAME [SQLCLREINVOICE].[StoredProcedures].[InsertarDocumentos]
GO
```

### 5. Crear un linked server <a name="cinco"></a>
Se debe crear un nuevo linked server con POS-CS-CARTAGO (o cualquier otro servidor previamente configurado y que cuente con el super-usario **giprod** en la base de datos) para que a la hora de ejecutar los scripts de creación de tablas, se importen los datos que existen en la nueva instancia del servidor.
Para crear el Linked Server, siga los siguientes pasos:

    |-- Server Objects
        |-- Linked Servers
            |-- New Linked Server
                |-- General
                    |-- Linked Server: POS-CS-CARTAGO
                    |-- Serve Type: SQL Server
                |-- Security
                    |-- Check: Be made using this security context
                    |-- Remote login: giprod
                    |-- With password: UserPassword

### 6. Creación de tablas de parámetros <a name="seis"></a>
Ejecute los siguientes scripts para crear las tablas de parámetros:

```SQL
SELECT * 
INTO EINVOICECURRENCYCR
FROM [POS-CS-CARTAGO].[RETAILSTORE].[DBO].[EINVOICECURRENCYCR]

SELECT * 
INTO EINVOICEPARAMETERSCR
FROM [POS-CS-CARTAGO].[RETAILSTORE].[DBO].[EINVOICEPARAMETERSCR]

SELECT * 
INTO EINVOICEPAYMMODECR
FROM [POS-CS-CARTAGO].[RETAILSTORE].[DBO].[EINVOICEPAYMMODECR]
```

### 7. Creación de la vista de identificaciones <a name="siete"></a>
Ejecute el siguiente script para crear la vista que contendrá la relación de la identificación de cada cliente con el tipo de identificación que le corresponda según lo solicitado por le Ministerio de Hacienda de Costa Rica:

```SQL
GO

-- =============================================
-- Author:          Jeffrey Camareno Fonseca
-- Create date:     2018-09-10
-- Description:	    Relaciona cada Vatnum con el tipo de identificacion correspondiente
-- =============================================
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE VIEW [crt].[RETAILVATNUMIDVIEW] AS
(
SELECT  A.RECID, A.INSTANCERELATIONTYPE, C.ACCOUNTNUM, A.NAME,C.VATNUM, Left(RIGHT((c.vatnum),5),1) VALIDATOR, REPLACE(c.VATNUM, '-', '') IDFORMATED, LEN(REPLACE(c.VATNUM, '-', '')) VATNUMLEN, 
		CASE
			WHEN 
				Left(RIGHT((c.vatnum),5),1)		=	'-'	AND	
				LEN(REPLACE(c.vatnum, '-', '')) =	9	AND 
				Left(c.vatnum,1)				<>	'0'	AND 
				c.vatnum LIKE '[0-9]%'
				THEN	'1'		--cedula fisica
			
			WHEN
				Left(RIGHT((c.vatnum),7),1)		=	'-'	AND
				Left(c.vatnum,1)				=	'3'	AND 
				LEN(REPLACE(c.vatnum, '-', '')) =	10	AND 
				Left(c.vatnum,1)				<>	'0'	AND 
				c.vatnum LIKE '[0-9]%'
				THEN	'2'		--cedula juridica

			WHEN 
				LEN(REPLACE(c.vatnum, '-', ''))	=	11	OR 
				LEN(REPLACE(c.vatnum, '-', '')) =	12	AND
				Left(c.vatnum,1)				<>	'0'	AND 
				c.vatnum LIKE '[0-9]%'
				THEN	'3'		--dimex 

			WHEN
				Len(REPLACE(c.vatnum,'-',''))	=	10	AND 
				Left(c.vatnum,1)				<>	'0'	AND 
				c.vatnum LIKE '[0-9]%'	
				THEN	'4'		-- nite

			WHEN
				Len(REPLACE(c.vatnum,'-',''))	=	0
				THEN	'98'	-- Error: sin cedula	

			ELSE		'10'	-- Exranjero
		END as 'IDTYPE'

  FROM DIRPARTYTABLE A, CUSTTABLE C
	WHERE A.PARTYNUMBER		= C.ACCOUNTNUM
		AND C.DATAAREAID	='FMCM'
)

GO
```

### 8. Funciones <a name="ocho"></a>
Ejecute los siguientes scripts para crear las siguientes funciones:

- [getCustomerName]

```SQL
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[getCustomerName]
(
       -- Add the parameters for the function here
       @AccountNum varchar(20)
)
RETURNS varchar(100)
AS
BEGIN
		-- Declare the return variable here
		DECLARE @FullName varchar(100)

		-- Add the T-SQL statements to compute the return value here
		SELECT @FullName = ltrim(rtrim(REPLACE(NAME,char(9),''))) 
		FROM CUSTTABLE CUST
		INNER JOIN DIRPARTYTABLE DPT		
			ON CUST.PARTY = DPT.RECID
		WHERE CUST.ACCOUNTNUM = @AccountNum
		--DIRPARTYTABLE WHERE PARTYNUMBER = @AccountNum)
		SET @FullName = REPLACE(@FullName, '&', '')
		SET @FullName = REPLACE(@FullName, '“', '')
		SET @FullName = REPLACE(@FullName, '‘', '')
		SET @FullName = REPLACE(@FullName, '<', '')
		SET @FullName = REPLACE(@FullName, '>c', '')
       -- Return the result of the function
       RETURN ISNULL(@FullName,'Cliente Genérico')
END
```

- [getCurrencyCode]

```SQL
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[getCurrencyCode]
(
       -- Add the parameters for the function here
       @CURRENCYCODEAX nvarchar(3)
)
RETURNS INT
AS
BEGIN
       -- Declare the return variable here
       DECLARE @CODCURRENCYHACIENDA INT

       -- Add the T-SQL statements to compute the return value here
       SET @CODCURRENCYHACIENDA = (SELECT CODCURRENCYHACIENDA FROM EINVOICECURRENCYCR WHERE CURRENCYCODEAX = @CURRENCYCODEAX)

       -- Return the result of the function
       RETURN @CODCURRENCYHACIENDA

END
```

- [getAccountNum]

```SQL
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[getAccountNum]
(
       -- Add the parameters for the function here
       @DataAreaId NVARCHAR(4)
)
RETURNS NVARCHAR(20)
AS
BEGIN
       -- Declare the return variable here
       DECLARE @ACCOUNTNUM NVARCHAR(20)


       -- Add the T-SQL statements to compute the return value here
       SET @ACCOUNTNUM = (SELECT ACCOUNTNUM FROM EINVOICEPARAMETERSCR WHERE DATAAREAID = @DataAreaId)

       -- Return the result of the function
       RETURN @ACCOUNTNUM

END
```

- [Get_TaxValuePercent]

```SQL
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[Get_TaxValuePercent]
(
	@DataAreaId		VARCHAR(5),
	@TaxItemGroup	VARCHAR(15)	
)
RETURNS REAL
AS      
BEGIN     
	DECLARE @TaxValue REAL
	
	SELECT @TaxValue = SUM(TD.TAXVALUE)
	FROM ax.TAXONITEM TOI
	INNER JOIN ax.TAXTABLE TB
		ON TOI.TAXCODE = TB.TAXCODE
		AND TOI.DATAAREAID = TB.DATAAREAID
	INNER JOIN ax.TAXDATA TD
		ON TB.TAXCODE = TD.TAXCODE
		AND TB.DATAAREAID = TD.DATAAREAID
	WHERE TOI.TAXITEMGROUP = @TaxItemGroup
		AND TOI.DATAAREAID = @DataAreaId

	RETURN  ISNULL(@TaxValue,0)	
   
END
```

- [Get_ItemName]

```SQL
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[Get_ItemName]
(
	@DataAreaId  VARCHAR(5),
	@ItemId VARCHAR(30) 
)
RETURNS VARCHAR(100)
AS      
BEGIN     
   -- 
   DECLARE @Name VARCHAR(500)
  
    SELECT TOP 1 @Name = ltrim(rtrim(REPLACE(ERPT.NAME,char(9),'')))   
	FROM ax.ECORESPRODUCTTRANSLATION ERPT
    JOIN INVENTTABLE IT 
		ON ERPT.product = IT.product		
		--AND ERPT.PARTITION = IT.PARTITION
	WHERE  IT.dataareaid = @dataareaid 
		and IT.itemid  = @itemid 
		--AND IT.PARTITION = 5637144576   
   SET @Name = REPLACE(@Name, '&', '')
   SET @Name = REPLACE(@Name, '“', '')
   SET @Name = REPLACE(@Name, '‘', '')
   SET @Name = REPLACE(@Name, '<', '')
   SET @Name = REPLACE(@Name, '>c', '')
   --
   RETURN ISNULL(@Name,'No definida')	
   --
END
```

- [getCustomerVatNum]

```SQL
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- Author:		Jeffrey Camareno
-- Create date: 28/09/18
-- Description:	<Retorna el numero de cedula dado el accountnum
-- =============================================
CREATE FUNCTION [dbo].[getCustomerVatNum]
(@AccountNum varchar(20)) --PARAMETRO

RETURNS varchar(20)
AS
BEGIN
	-- Declare the return variable here
	DECLARE @VatNumFormatted varchar(20)

	-- Add the T-SQL statements to compute the return value here
	set @VatNumFormatted = (SELECT IdFormated FROM crt.RETAILVATNUMIDVIEW where ACCOUNTNUM = @AccountNum)
	
	-- Return the result of the function
	RETURN @VatNumFormatted

END

GO
```

- [getCustomerIdType]

```SQL
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- Author:		Jeffrey Camareno
-- Create date: 28/09/18
-- Description:	<Retorna el codigo del tipo de ID dado el accountnum
-- =============================================

CREATE FUNCTION [dbo].[getCustomerIdType]
(@AccountNum varchar(20)) --PARAMETRO

RETURNS varchar(20)
AS
BEGIN
	-- Declare the return variable here
	DECLARE @IDtype varchar(20)

	-- Add the T-SQL statements to compute the return value here
	set @IDtype = (SELECT IDTYPE FROM crt.RETAILVATNUMIDVIEW where ACCOUNTNUM = @AccountNum)
	
	-- Return the result of the function
	RETURN @IDtype

END

GO
```

- [getCustomerEmail]

```SQL
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		Jeffrey Camareno
-- Create date: 05/11/18
-- Description:	Returna el email asociado al cliente
-- =============================================
CREATE FUNCTION [dbo].[getCustomerEmail]
(@AccountNum varchar(20)) --PARAMETRO

RETURNS varchar(50)
AS
BEGIN
	-- Declare the return variable here
	DECLARE @custEmail varchar(50)

	-- Add the T-SQL statements to compute the return value here
	set @custEmail = (SELECT email FROM crt.CUSTOMERSVIEW  where crt.CUSTOMERSVIEW.ACCOUNTNUMBER = @AccountNum)
	
	-- Return the result of the function
	RETURN @custEmail
END
```



### 9. Crear tabla de logs <a name="nueve"></a>
La tabla EINVOICELOG almacena las facturas procesadas en los POS junto con su estado actual respecto a la respuesta que obtuvo ante el Ministerio de Hacienda de Costa Rica.

Para crear la tabla ejecute el siguiente script:

```SQL
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[EINVOICELOG](
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[DATAAREAID] [nvarchar](4) NOT NULL DEFAULT ('fmcm'),
	[EINVOICEID] [nvarchar](50) NULL DEFAULT (''),
	[EINVOICESEQUENCEKEY] [nvarchar](50) NULL DEFAULT (''),
	[ERRORMSG] [nvarchar](255) NULL DEFAULT (''),
	[XMLSENDED] [nvarchar](max) NOT NULL DEFAULT (NULL),
	[XMLRECEIVED] [nvarchar](max) NOT NULL DEFAULT (NULL),
	[INVOICEID] [nvarchar](50) NOT NULL DEFAULT (''),
	[ERRORCODE] [nvarchar](20) NULL DEFAULT (''),
	[GI_SENTSITUATION] [int] NOT NULL DEFAULT ((1)),
	[STORE] [nvarchar](10) NULL DEFAULT (''),
	[TERMINAL] [nvarchar](10) NULL DEFAULT (''),
	[GI_EINVOICEINSERTSTATUS] [int] NOT NULL DEFAULT ((0)),
	[EINVOICETYPE] [int] NOT NULL,
	[RECEIPTID] [nvarchar](50) NOT NULL DEFAULT (''),
	[POSCREATEDDATE]  [datetime] NOT NULL DEFAULT (dateadd(millisecond, -datepart(millisecond,getutcdate()),getutcdate())),
	[CREATEDDATETIME] [datetime] NOT NULL DEFAULT (dateadd(millisecond, -datepart(millisecond,getutcdate()),getutcdate())),
 CONSTRAINT [PK_INVOICEBYDATE] PRIMARY KEY CLUSTERED 
(
	[CREATEDDATETIME] ASC,
	[INVOICEID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
```

### 10. Crear Store Procedure (SP) Insert_EInvoice (version 4.2) (NO VIGENTE)<a name="diez"></a>
El SP Insert_EInvoice es el que se encarga de contruir el XML de la factura que se va a Insertar en el Ministerio de Hacienda.

Para crear este SP ejecute el siguiente script:

```SQL
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- Author:		Jairo Martínez Ureña
-- Create date: 2018-08-09
-- Description:	Insertar documento electrónico
-- =============================================
-- Example: EXEC dbo.Insert_EInvoice 'fmcm', '0003-000015-42931'

CREATE PROCEDURE [dbo].[Insert_EInvoice]
	@DATAAREAID VARCHAR(5),
	@TRANSACTIONID VARCHAR(50)
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	SET NOCOUNT ON;
	
	--SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
-----------------------------
----PARAMETROS NODO DE ENCABEZADO
--------------------------
		CREATE TABLE #EncabezadoXML( 
		NumCuenta					VARCHAR(20),
		TipoDoc						INT,
		FechaFactura				VARCHAR(19),
		TipoCambio					INT,
		CondicionVenta				INT,
		IdMedioPago					INT,
		Sucursal					INT,
		Terminal					INT,
		Moneda						INT,
		SituacionEnvio				INT,
		TransactionId				VARCHAR(50),
		ReceiptId					VARCHAR(50),
		NombreReceptor				VARCHAR(200),
		TipoIdentificacion			VARCHAR(2),		--NUEVO
		IdentificacionReceptor		VARCHAR(20),	--NUEVO
		CorreoElectronicoReceptor	VARCHAR(200),
		CopiaCortesia				VARCHAR(200)
	)
	
	-- TOMA EL # DE FACTURA
	SELECT *
	INTO #RETTRANSTMP
	FROM AX.RETAILTRANSACTIONTABLE WITH (NOLOCK)
	WHERE TRANSACTIONID = @TRANSACTIONID

	--PARAMETROS
	DECLARE @NUMCUENTA			NVARCHAR(10)
	DECLARE @PASS				NVARCHAR(30)
	DECLARE @EMAIL				NVARCHAR(200)	
	DECLARE @SUC				NVARCHAR(3)
	DECLARE @TERM				NVARCHAR(3)
	DECLARE @EINVOICETYPE		INT;
	DECLARE @TRANSACTION		NVARCHAR(50)
	DECLARE @RECEIPT			NVARCHAR(50)
	DECLARE @NUMBEROFITEMS		NUMERIC(32,16)
	DECLARE @COMPANYEMAIL		NVARCHAR(200)
	DECLARE @POSDATETIME		DATETIME
	
	SELECT 
		@RECEIPT				= RT.RECEIPTID,
		@SUC					= CONVERT(NVARCHAR(3),CONVERT(INT, RT.STORE)),
		@TERM					= CONVERT(NVARCHAR(3),CONVERT(INT, RT.TERMINAL))	,	
		@EINVOICETYPE			= CASE WHEN SALEISRETURNSALE = 1 OR PAYMENTAMOUNT < 0 THEN 3 ELSE 4 END,
		@TRANSACTION			= RT.TRANSACTIONID,
		@NUMBEROFITEMS			= RT.NUMBEROFITEMS,
		@POSDATETIME			= DATEADD(HH, -6, RT.CREATEDDATETIME)
	FROM #RETTRANSTMP RT

	SELECT 
		@NUMCUENTA				= PARM.ACCOUNTNUM,
		@PASS					= PARM.PASSWORD, 
		@EMAIL					= PARM.USERADMINEMAIL,
		@COMPANYEMAIL			= PARM.SALESEMAIL
	FROM EINVOICEPARAMETERSCR PARM
	WHERE PARM.DATAAREAID		= @DATAAREAID

	--NODO ENCABEZADO
	--BEGIN TRAN 
	INSERT INTO #EncabezadoXML
		SELECT TOP 1 
			NumCuenta					= @NUMCUENTA, --dbo.getAccountNum(RT.DATAAREAID),
			TipoDoc						= @EINVOICETYPE,
			FechaFactura				= CONVERT(VARCHAR(19),DATEADD(HH, -6, RT.CREATEDDATETIME),126),
			TipoCambio					= CONVERT(INT,(RT.EXCHRATE/100)),
			CondicionVenta				= 1,
			IdMedioPago					= 1,--RTP.TENDERTYPE, 
			Sucursal					= CONVERT(INT, RT.STORE),
			Terminal					= CONVERT(INT, RT.TERMINAL),
			Moneda						= dbo.getCurrencyCode(RT.CURRENCY),
			SituacionEnvio				= 1,			
			TransactionId				= @TRANSACTION,
			ReceiptId					= @RECEIPT ,
			NombreReceptor				= dbo.getCustomerName(RT.CUSTACCOUNT),
			TipoIdentificacion			= dbo.getCustomerIdType(RT.CUSTACCOUNT),
			IdentificacionReceptor		= dbo.getCustomerVatNum(RT.CUSTACCOUNT),
			CorreoElectronicoReceptor	= dbo.getCustomerEmail(RT.CUSTACCOUNT),
			CopiaCortesia				= @COMPANYEMAIL
		FROM #RETTRANSTMP	RT
		INNER JOIN RETAILTRANSACTIONPAYMENTTRANS RTP (NOLOCK)
			ON RTP.RECEIPTID	= RT.RECEIPTID
			AND RTP.DATAAREAID	= RT.DATAAREAID		
	--COMMIT TRAN
	-- FIN NODO ENCABEZADO
----------------------------
----PARAMETROS NODO DE DETALLE
--------------------------
	DECLARE @DETROWNCOUNT INT	
	
	CREATE TABLE #DetalleXML( 
		Tipo						VARCHAR(1),
		CodigoProducto				VARCHAR(20),
		Cantidad					DECIMAL(15,2),
		UnidadMedida				INT,
		DetalleMerc					VARCHAR(70),
		PrecioUnitario				DECIMAL(15,2),	
		MontoDescuento				DECIMAL(15,2),
		NaturalezaDescuento			VARCHAR(100),	
		CodigoImpuesto				INT,
		PorcentajeImpuesto			DECIMAL(6,2),
		MontoImpuesto				DECIMAL(15,2),
		PrecioBruto					DECIMAL(15,2),
		TaxItemGroup				VARCHAR(30),
		TransactionId				VARCHAR(50),
		ReceiptId					VARCHAR(20),
		ReturnTransactionId			VARCHAR(50))
	
	--Nodo Detalle
	--BEGIN TRAN 
	INSERT INTO #DetalleXML
		SELECT
			Tipo					= CASE IT.ITEMTYPE WHEN 0 THEN 'M' WHEN 2 THEN 'S' END,
			CodigoProducto			= RTSALES.ITEMID,
			Cantidad				= CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.QTY ELSE RTSALES.QTY *-1  END, 
			UnidadMedida			= 95,
			DetalleMerc				= dbo.Get_ItemName(RTSALES.DATAAREAID, RTSALES.ITEMID),
			PrecioUnitario			= RTSALES.PRICE,
			MontoDescuento			= CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.DISCAMOUNT*-1 ELSE RTSALES.DISCAMOUNT  END, 
			NaturalezaDescuento		= CASE WHEN RTSALES.DISCAMOUNT <> 0 THEN 'Descuento por promoción' ELSE '' END,	
			CodigoImpuesto			= CASE RTSALES.TAXITEMGROUP WHEN 'EXENTOS' THEN '' ELSE '1' END,
			PorcentajeImpuesto		= dbo.Get_TaxValuePercent(RTSALES.DATAAREAID, RTSALES.TAXITEMGROUP),
			MontoImpuesto			= CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.TAXAMOUNT ELSE RTSALES.TAXAMOUNT *-1  END,
			PrecioBruto				= RTSALES.PRICE * (CASE WHEN SALEISRETURNSALE = 1 OR PAYMENTAMOUNT  < 0 THEN RTSALES.QTY ELSE RTSALES.QTY *-1 END),
			TaxItemGroup			= RTSALES.TAXITEMGROUP,
			TransactionId			= RT.TRANSACTIONID,
			ReceiptId				= RT.RECEIPTID,
			ReturnTransactionId		= RTSALES.RETURNTRANSACTIONID
		FROM RETAILTRANSACTIONSALESTRANS RTSALES (NOLOCK)
		INNER JOIN #RETTRANSTMP RT
			ON RT.TRANSACTIONID		= RTSALES.TRANSACTIONID
			AND RT.STORE			= RTSALES.STORE
			AND RT.TERMINAL			= RTSALES.TERMINALID
			AND RT.DATAAREAID		= RTSALES.DATAAREAID
		INNER JOIN ax.INVENTTABLE IT
			ON IT.ITEMID			= RTSALES.ITEMID
			AND IT.DATAAREAID		= IT.DATAAREAID
			AND RTSALES.TRANSACTIONSTATUS = 0 -- Solo los que se facturaron

		SET @DETROWNCOUNT = @@ROWCOUNT;

		-- ACTUALIZA MONTO IMPUESTO (TEMAS DE HACIENDA)
		UPDATE #DetalleXML
			SET MontoImpuesto		= CONVERT(DECIMAL(15,2),((PrecioUnitario * Cantidad) - MontoDescuento) * PorcentajeImpuesto/100)
		WHERE @EINVOICETYPE			= 3

	--COMMIT TRAN
	-- FIN NODO DETALLE	
		
----------------------------
----PARAMETROS NODO DE REFERENCIA (Devolución)
--------------------------
	CREATE TABLE #ReferenciaXML( 
		TpoDocRef				VARCHAR(2),
		NumeracionReferencia	VARCHAR(50), -- Tiquete electrónico o clave numerica
		CodigoReferencia		VARCHAR(2),
		TransactionId			VARCHAR(50),
		ReceiptId				VARCHAR(20))
	
	IF(@EINVOICETYPE = 3)
	BEGIN 
		DECLARE @ReturnTransactionId VARCHAR(50)
		
		SET	@ReturnTransactionId		= (SELECT TOP 1 ReturnTransactionId FROM #DetalleXML)
		
		INSERT INTO #ReferenciaXML
			SELECT TpoDocRef			= '04',
				NumeracionReferencia	= EINV.EINVOICEID, --@ReturnTransactionId,
				CodigoReferencia		= CASE WHEN @NUMBEROFITEMS = TRANS.NUMBEROFITEMS THEN '01' ELSE '03' END, -- 1 anula referencia , 2 corrige texto, 3 corrige monto
				TRANS.TransactionId, 
				TRANS.ReceiptId 
			FROM RETAILTRANSACTIONTABLE TRANS (NOLOCK)	
			INNER JOIN EINVOICELOG EINV
				ON EINV.INVOICEID		= TRANS.TransactionId
				AND EINV.STORE			= CONVERT(INT,TRANS.STORE)
				AND EINV.EINVOICEID IS NOT NULL
			WHERE TRANS.DATAAREAID		= @DATAAREAID
				AND TRANS.TRANSACTIONID = @ReturnTransactionId			
	END
	-- FIN NODO REFERENCIA		
----------------------------
----PARAMETROS NODO DE TOTALES
--------------------------
	DECLARE @TotalServGravados			DECIMAL(15,2)
	DECLARE	@TotalServExentos			DECIMAL(15,2)
	DECLARE	@TotalMercanciasGravadas	DECIMAL(15,2)
	DECLARE	@TotalMercanciasExentas		DECIMAL(15,2)
	DECLARE	@TotalGravado				DECIMAL(15,2)
	DECLARE	@TotalExento				DECIMAL(15,2)
	DECLARE	@TotalVenta					DECIMAL(15,2)
	DECLARE	@TotalDescuentos			DECIMAL(15,2)
	DECLARE	@TotalVentaNeta				DECIMAL(15,2)
	DECLARE	@TotalImpuesto				DECIMAL(15,2)
	DECLARE	@TotalComprobante			DECIMAL(15,2)
	--DECLARE @NumeroFactura			VARCHAR(20)

	CREATE TABLE #TotalesXML( 
		TotalServGravados				DECIMAL(15,2),
		TotalServExentos				DECIMAL(15,2),
		TotalMercanciasGravadas			DECIMAL(15,2),
		TotalMercanciasExentas			DECIMAL(15,2),
		TotalGravado					DECIMAL(15,2),
		TotalExento						DECIMAL(15,2),
		TotalVenta						DECIMAL(15,2),
		TotalDescuentos					DECIMAL(15,2),
		TotalVentaNeta					DECIMAL(15,2),
		TotalImpuesto					DECIMAL(15,2),
		TotalComprobante				DECIMAL(15,2),
		TransactionId					VARCHAR(50),
		ReceiptId						VARCHAR(20))
	
	SELECT 
		@TotalServGravados				= ISNULL(SUM(F.TotalServGravados),0),
		@TotalServExentos				= ISNULL(SUM(F.TotalServExentos),0),
		@TotalMercanciasGravadas		= ISNULL(SUM(F.TotalMercanciasGravadas),0),
		@TotalMercanciasExentas			= ISNULL(SUM(F.TotalMercanciasExentas),0),
		@TotalGravado					= @TotalServGravados + @TotalMercanciasGravadas,  
		@TotalExento					= @TotalServExentos + @TotalMercanciasExentas, 
		@TotalVenta						= @TotalGravado + @TotalExento,
		@TotalDescuentos				= SUM(F.TotalDescuentos),
		@TotalVentaNeta					= @TotalVenta - @TotalDescuentos,
		@TotalImpuesto					= SUM(F.TotalImpuesto),
		@TotalComprobante				= @TotalVentaNeta + @TotalImpuesto
	FROM (
		SELECT 
			TotalServGravados			= CASE WHEN Tipo  = 'S' AND TaxItemGroup <> 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalServExentos			= CASE WHEN Tipo  = 'S' AND TaxItemGroup = 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalMercanciasGravadas		= CASE WHEN Tipo  = 'M' AND TaxItemGroup <> 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalMercanciasExentas		= CASE WHEN Tipo  = 'M' AND TaxItemGroup = 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalDescuentos				= SUM(MontoDescuento),
			TotalImpuesto				= SUM(MontoImpuesto),
			TransactionId, ReceiptId
		FROM #DetalleXML
		GROUP BY Tipo, TransactionId, ReceiptId, TaxItemGroup ) F
	GROUP BY F.TransactionId, F.ReceiptId

	--BEGIN TRAN 
	--Nodo Totales
	INSERT INTO #TotalesXML
		SELECT 
			@TotalServGravados,
			@TotalServExentos,
			@TotalMercanciasGravadas,
			@TotalMercanciasExentas,
			@TotalGravado,  
			@TotalExento,
			@TotalVenta	,
			@TotalDescuentos,
			@TotalVentaNeta,
			@TotalImpuesto,
			@TotalComprobante,
			@TRANSACTION,
			@RECEIPT
			
	-- FIN NODO TOTALES	
	--COMMIT TRAN
	--CREACION XML	
	DECLARE	@XMLString	VARCHAR(MAX)	
	 
	SET @XMLString =  (	
		SELECT 
			(SELECT 
				TransactionId,
				FechaFactura,	
				(SELECT NumCuenta FOR	XML PATH('Emisor'), TYPE),
				TipoCambio,
				TipoDoc,			
				CondicionVenta,
				Moneda,
				IdMedioPago,		
				Sucursal,
				Terminal,	
				SituacionEnvio,
				( SELECT NombreReceptor, TipoIdentificacion, IdentificacionReceptor, CorreoElectronicoReceptor, CopiaCortesia FOR XML PATH('Receptor'), TYPE)
			FROM #EncabezadoXML
			FOR	XML PATH('Encabezado'),TYPE),
			(SELECT
				(SELECT 
				Tipo,
				CodigoProducto,
				Cantidad,
				UnidadMedida,
				DetalleMerc,
				PrecioUnitario,
				MontoDescuento,
				NaturalezaDescuento,
				(SELECT CodigoImpuesto,PorcentajeImpuesto, MontoImpuesto FOR	XML PATH('Impuesto'), TYPE, ROOT ('Impuestos'))			
				FROM #DetalleXML
				FOR	XML PATH('Linea'),TYPE)
			FOR XML PATH('Detalle'),TYPE),		
				(SELECT TpoDocRef = ISNULL(TpoDocRef,0),	NumeracionReferencia = ISNULL(NumeracionReferencia,0), CodigoReferencia = ISNULL(CodigoReferencia,0)
					FROM #ReferenciaXML
				FOR	XML PATH('Referencia'), TYPE, ROOT ('InformacionDeReferencia')),
			(SELECT 
				TotalServGravados,
				TotalServExentos,
				TotalMercanciasGravadas,
				TotalMercanciasExentas,
				TotalGravado,  
				TotalExento,
				TotalVenta	,
				TotalDescuentos,
				TotalVentaNeta,
				TotalImpuesto,
				TotalComprobante
			FROM #TotalesXML
			FOR	XML PATH('Totales'),TYPE)
		FOR XML PATH('FacturaElectronicaXML'), ROOT('root')
		)

	DECLARE	@XMLPrefix	VARCHAR(100)	
	SET @XMLPrefix = '<?xml version="1.0" encoding="UTF-8"?>'
	
	SET @XMLString = @XMLPrefix + @XMLString

	DECLARE @INVOICE_EXIST NVARCHAR(20)

	SELECT @INVOICE_EXIST =  INVOICEID
	FROM EINVOICELOG
	WHERE INVOICEID = @TRANSACTION
		AND EINVOICEID <> ''

	IF(@INVOICE_EXIST IS NULL)
	BEGIN

		--SELECT XMLstr = @XMLString, Email = @EMAIL, Pwd = @PASS, Factura = @TRANSACTION, Sucursal = @SUC, Term = @TERM
		EXEC dbo.InsertarDocumentosEInvoice @XMLString, @EMAIL, @PASS, @TRANSACTION, @SUC, @TERM
	
		--Inserta el XML en una tabla temporal para recorrerlo
		CREATE TABLE #XMLwithOpenXML (
			Id				INT IDENTITY PRIMARY KEY,
			XMLData			XML,
			LoadedDateTime  DATETIME )

		WAITFOR DELAY '00:00:02' 

		DECLARE @RetailStore	Varchar(10);
		DECLARE @XmlRespPath	Varchar(100)		= 'C:\\FacturaElectronica\Homex\DocumentosInsertados\'
		DECLARE @FileName		Varchar(100);
		DECLARE @Ext			Varchar(6)			= '.xml';
		DECLARE @Sufix			Varchar(11)			= '_respuesta'
		DECLARE @sql			nvarchar(MAX);

		SET @RetailStore	= '_' +  @SUC +'_' + @TERM	; 

		SET @FileName		= @XmlRespPath  + @TRANSACTION +  @RetailStore + @Sufix + @ExT;
		
		Select @sql			= N'INSERT INTO #XMLwithOpenXML(XMLData, LoadedDateTime)
					SELECT CONVERT(XML, BulkColumn) AS BulkColumn, GETDATE() 
				FROM OPENROWSET(BULK ''' + @FileName + ''', SINGLE_BLOB) AS x;'
	
		EXEC sp_executesql @sql
	
		DECLARE @XML AS XML, @hDoc AS INT --, @SQL NVARCHAR (MAX)
		DECLARE @EINVOICEID		NVARCHAR(30)
		DECLARE @SEQUENCENUM	NVARCHAR(50)
		DECLARE @ERRORCODE		NVARCHAR(20)
		DECLARE @ERRORMSG		NVARCHAR(255)
		DECLARE @XMLRECEIVEDSTR NVARCHAR(MAX)

		SELECT @XML = XMLData, @XMLRECEIVEDSTR = convert(varchar(max),XMLData )
		FROM #XMLwithOpenXML	
	
		EXEC sp_xml_preparedocument @hDoc OUTPUT, @XML

		SELECT 
			@SEQUENCENUM	= ClaveNumerica ,
			@EINVOICEID		= NumConsecutivoCompr,
			@ERRORCODE		= CodigoError,
			@ERRORMSG		= DescripcionError
		FROM OPENXML(@hDoc, 'root/FacturaElectronicaXML')
		WITH 
		(	
			ClaveNumerica [nvarchar](50)  'ClaveNumerica',
			NumConsecutivoCompr [nvarchar](30) 'NumConsecutivoCompr',
			CodigoError [varchar](20) 'CodigoError',		
			DescripcionError [varchar](255) 'DescripcionError'	
		)

		EXEC sp_xml_removedocument @hDoc
	 	
		--BEGIN TRY	
		--BEGIN TRAN
		--SELECT @RECEIPTID
		
		BEGIN TRAN	
			DELETE FROM EINVOICELOG WHERE INVOICEID = @TRANSACTION
		COMMIT
		
		INSERT INTO EINVOICELOG (DATAAREAID, EINVOICEID, EINVOICESEQUENCEKEY, ERRORMSG, XMLSENDED, XMLRECEIVED, INVOICEID, ERRORCODE, GI_SENTSITUATION, STORE, TERMINAL, GI_EINVOICEINSERTSTATUS, EINVOICETYPE, CREATEDDATETIME, RECEIPTID, POSCREATEDDATE)
								VALUES (@DATAAREAID, ISNULL(@EINVOICEID, ''), ISNULL(@SEQUENCENUM,''), @ERRORMSG, @XMLString,@XMLRECEIVEDSTR, @TRANSACTION, @ERRORCODE, 1, @SUC, @TERM, 0, @EINVOICETYPE,DATEADD(HOUR,-6,GETUTCDATE()), @RECEIPT,  @POSDATETIME)
		--COMMIT TRAN
	END
	
	DROP TABLE #RETTRANSTMP
	DROP TABLE #EncabezadoXML
	DROP TABLE #DetalleXML
	DROP TABLE #TotalesXML
	DROP TABLE #XMLwithOpenXML
-- Example: EXEC dbo.Insert_EInvoice 'fmcm', '00030005175859'

END

```

### 11. Crear Store Procedure (SP) Insert_EInvoice (version 4.3) (VIGENTE)<a name="once"></a>
El SP Insert_EInvoice es el que se encarga de contruir el XML de la factura que se va a Insertar en el Ministerio de Hacienda, en la versión 4.3 (vigente a partir del 01 de Julio de 2019).

Para crear este SP ejecute el siguiente script:

```SQL
USE [RetailStore]
GO
/****** Object:  StoredProcedure [dbo].[Insert_EInvoice]    Script Date: 06/30/2019 22:30:32 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- Author:		JEFFREY CAMARENO FONSECA
-- Description:	Insertar documento electrónico
-- =============================================
-- Example: EXEC dbo.Insert_EInvoice 'fmcm', '0003-000015-42931'

CREATE PROCEDURE [dbo].[Insert_EInvoice]
	@DATAAREAID VARCHAR(5),
	@TRANSACTIONID VARCHAR(50)
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	SET NOCOUNT ON;
	
	--SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
-----------------------------
----PARAMETROS NODO DE ENCABEZADO
--------------------------
		CREATE TABLE #EncabezadoXML( 
		NumCuenta					VARCHAR(20),
		TipoDoc						INT,
		FechaFactura				VARCHAR(19),
		TipoCambio					INT,
		CondicionVenta				INT,
		idMedioPago					INT,
		Sucursal					INT,
		Terminal					INT,
		Moneda						INT,
		SituacionEnvio				INT,
		TransactionId				VARCHAR(50),
		ReceiptId					VARCHAR(50),
		NombreReceptor				VARCHAR(200),
		TipoIdentificacion			VARCHAR(2),		--NUEVO
		IdentificacionReceptor		VARCHAR(20),	--NUEVO
		CodigoActividad				VARCHAR(6),     --Version  4.3
		CorreoElectronicoReceptor	VARCHAR(200),
		CopiaCortesia				VARCHAR(200)
	)
	
	-- TOMA EL # DE FACTURA
	SELECT *
	INTO #RETTRANSTMP
	FROM AX.RETAILTRANSACTIONTABLE WITH (NOLOCK)
	WHERE TRANSACTIONID = @TRANSACTIONID

	--PARAMETROS
	DECLARE @NUMCUENTA			NVARCHAR(10)
	DECLARE @CodigoActividad	NVARCHAR(6)
	DECLARE @PASS				NVARCHAR(30)
	DECLARE @EMAIL				NVARCHAR(200)	
	DECLARE @SUC				NVARCHAR(3)
	DECLARE @TERM				NVARCHAR(3)
	DECLARE @EINVOICETYPE		INT;
	DECLARE @EINVOICETYPETMP    INT;
	DECLARE @TRANSACTION		NVARCHAR(50)
	DECLARE @RECEIPT			NVARCHAR(50)
	DECLARE @NUMBEROFITEMS		NUMERIC(32,16)
	DECLARE @COMPANYEMAIL		NVARCHAR(200)
	DECLARE @POSDATETIME		DATETIME
	
	if exists(select 1 FROM DIRPARTYTABLE A, CUSTTABLE C
	WHERE A.PARTYNUMBER		= C.ACCOUNTNUM 
		AND C.DATAAREAID	='FMCM' and C.accountnum is not null and C.accountnum=(select top 1 RT.custaccount FROM #RETTRANSTMP	RT
		INNER JOIN RETAILTRANSACTIONPAYMENTTRANS RTP (NOLOCK)
			ON RTP.RECEIPTID	= RT.RECEIPTID
			AND RTP.DATAAREAID	= RT.DATAAREAID and dbo.getCustomerIdType(RT.CUSTACCOUNT)<>'10' ))
		begin
			set @EINVOICETYPETMP=1;
		end 
		else begin
			set @EINVOICETYPETMP=4;
		end
	
	SELECT 
		@RECEIPT				= RT.RECEIPTID,
		@SUC					= CONVERT(NVARCHAR(3),CONVERT(INT, RT.STORE)),
		@TERM					= CONVERT(NVARCHAR(3),CONVERT(INT, RT.TERMINAL))	,	
		@EINVOICETYPE			= CASE WHEN SALEISRETURNSALE = 1 OR PAYMENTAMOUNT < 0 THEN 3 ELSE @EINVOICETYPETMP END,
		@TRANSACTION			= RT.TRANSACTIONID,
		@NUMBEROFITEMS			= RT.NUMBEROFITEMS,
		@POSDATETIME			= DATEADD(HH, -6, RT.CREATEDDATETIME)
	FROM #RETTRANSTMP RT

	SELECT 
		@NUMCUENTA				= PARM.ACCOUNTNUM,
		@PASS					= PARM.PASSWORD, 
		@EMAIL					= PARM.USERADMINEMAIL,
		@COMPANYEMAIL			= PARM.SALESEMAIL,
		@CodigoActividad		= '521101' --PARM.ECONOMICACTIVITY   TODO Version  4.3
	FROM EINVOICEPARAMETERSCR PARM
	WHERE PARM.DATAAREAID		= @DATAAREAID
	
	--NODO ENCABEZADO
	--BEGIN TRAN 
	INSERT INTO #EncabezadoXML 
		SELECT TOP 1 
			NumCuenta					= @NUMCUENTA, --dbo.getAccountNum(RT.DATAAREAID),
			TipoDoc						= @EINVOICETYPE,
			FechaFactura				= CONVERT(VARCHAR(19),DATEADD(HH, -6, RT.CREATEDDATETIME),126),
			TipoCambio					= CONVERT(INT,(RT.EXCHRATE/100)),
			CondicionVenta				= 1,
			idMedioPago					= 1,--RTP.TENDERTYPE, 
			Sucursal					= CONVERT(INT, RT.STORE),
			Terminal					= CONVERT(INT, RT.TERMINAL),
			Moneda						= dbo.getCurrencyCode(RT.CURRENCY),
			SituacionEnvio				= 1,	
			TransactionId				= @TRANSACTION,
			ReceiptId					= @RECEIPT ,
			NombreReceptor				= dbo.getCustomerName(RT.CUSTACCOUNT),
			TipoIdentificacion			= dbo.getCustomerIdType(RT.CUSTACCOUNT),
			IdentificacionReceptor		= dbo.getCustomerVatNum(RT.CUSTACCOUNT),
			CodigoActividad				= @CodigoActividad,	 --Version  4.3	
			CorreoElectronicoReceptor	= dbo.getCustomerEmail(RT.CUSTACCOUNT),--CASE WHEN RT.RECEIPTEMAIL LIKE '%_@__%.__%'  AND PATINDEX('%[^a-z,0-9,@,.,_]%', REPLACE(RT.RECEIPTEMAIL, '-', 'a')) = 0 THEN RT.RECEIPTEMAIL ELSE @COMPANYEMAIL END, --CASE RT.RECEIPTEMAIL WHEN '' THEN 'vfactelectronic@grupointeca.com' ELSE RT.RECEIPTEMAIL END, --RECEIPTEMAIL
			CopiaCortesia				= ''--@COMPANYEMAIL
		FROM #RETTRANSTMP	RT
		INNER JOIN RETAILTRANSACTIONPAYMENTTRANS RTP (NOLOCK)
			ON RTP.RECEIPTID	= RT.RECEIPTID
			AND RTP.DATAAREAID	= RT.DATAAREAID		
	--COMMIT TRAN
	-- FIN NODO ENCABEZADO
----------------------------
----PARAMETROS NODO DE DETALLE
--------------------------

	DECLARE @DETROWNCOUNT INT	
	
	CREATE TABLE #DetalleXML( 
		Tipo						VARCHAR(1),
		CodigoProducto				VARCHAR(20),
		Cantidad					DECIMAL(15,2), --INT, cambio a decimal para que pueda vender pollo en terminos de kilos
		UnidadMedida				INT,
		DetalleMerc					VARCHAR(70),
		PrecioUnitario				DECIMAL(15,2),	
		MontoDescuento				DECIMAL(15,2),
		NaturalezaDescuento			VARCHAR(100),	
		CodigoImpuesto				INT,
		CodigoTarifa				INT,	--VERSION 4.3
		PorcentajeImpuesto			DECIMAL(6,2),
		MontoImpuesto				DECIMAL(15,2),
		PrecioBruto					DECIMAL(15,2),
		TaxItemGroup				VARCHAR(30),
		TransactionId				VARCHAR(50),
		ReceiptId					VARCHAR(20),
		ReturnTransactionId			VARCHAR(50))
	
	--Nodo Detalle
	--BEGIN TRAN 
	INSERT INTO #DetalleXML
		SELECT
			Tipo					= CASE IT.ITEMTYPE WHEN 0 THEN 'M' WHEN 2 THEN 'S' END,
			CodigoProducto			= RTSALES.ITEMID,
			Cantidad				= CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.QTY ELSE RTSALES.QTY *-1  END, 
			UnidadMedida			= 95,
			DetalleMerc				= dbo.Get_ItemName(RTSALES.DATAAREAID, RTSALES.ITEMID),
			PrecioUnitario			= RTSALES.PRICE,
			MontoDescuento			= CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.DISCAMOUNT*-1 ELSE RTSALES.DISCAMOUNT  END, 
			NaturalezaDescuento		= CASE WHEN RTSALES.DISCAMOUNT <> 0 THEN 'Descuento por promoción' ELSE '' END,	
			CodigoImpuesto			= CASE RTSALES.TAXITEMGROUP WHEN 'EXENTOS' THEN '' ELSE '1' END,
			CodigoTarifa			= CASE RTSALES.TAXITEMGROUP WHEN 'EXENTOS' THEN '' ELSE '8' END, --Version 4.3
			PorcentajeImpuesto		= dbo.Get_TaxValuePercent(RTSALES.DATAAREAID, RTSALES.TAXITEMGROUP),
			MontoImpuesto			= (RTSALES.PRICE * (CASE WHEN SALEISRETURNSALE = 1 OR PAYMENTAMOUNT  < 0 THEN RTSALES.QTY ELSE RTSALES.QTY *-1 END)-CASE WHEN RT.SALEISRETURNSALE = 1 OR RT.PAYMENTAMOUNT  < 0 THEN RTSALES.DISCAMOUNT*-1 ELSE RTSALES.DISCAMOUNT  END)*(dbo.Get_TaxValuePercent(RTSALES.DATAAREAID, RTSALES.TAXITEMGROUP)/100),
			PrecioBruto				= RTSALES.PRICE * (CASE WHEN SALEISRETURNSALE = 1 OR PAYMENTAMOUNT  < 0 THEN RTSALES.QTY ELSE RTSALES.QTY *-1 END),
			TaxItemGroup			= RTSALES.TAXITEMGROUP,
			TransactionId			= RT.TRANSACTIONID,
			ReceiptId				= RT.RECEIPTID,
			ReturnTransactionId		= RTSALES.RETURNTRANSACTIONID
		FROM RETAILTRANSACTIONSALESTRANS RTSALES (NOLOCK)
		INNER JOIN #RETTRANSTMP RT
			ON RT.TRANSACTIONID		= RTSALES.TRANSACTIONID
			AND RT.STORE			= RTSALES.STORE
			AND RT.TERMINAL			= RTSALES.TERMINALID
			AND RT.DATAAREAID		= RTSALES.DATAAREAID
		INNER JOIN ax.INVENTTABLE IT
			ON IT.ITEMID			= RTSALES.ITEMID
			AND IT.DATAAREAID		= IT.DATAAREAID
			AND RTSALES.TRANSACTIONSTATUS = 0 -- Solo los que se facturaron

		SET @DETROWNCOUNT = @@ROWCOUNT;

		-- ACTUALIZA MONTO IMPUESTO (TEMAS DE HACIENDA)
		--select @EINVOICETYPE,((cantidad*PrecioUnitario)-MontoDescuento)*(PorcentajeImpuesto/100),CONVERT(DECIMAL(15,2),round(((cantidad*PrecioUnitario)-MontoDescuento)*(PorcentajeImpuesto/100),2)),MontoImpuesto
		 --from #DetalleXML
		UPDATE #DetalleXML
			SET MontoImpuesto		= CONVERT(DECIMAL(15,2),round(((cantidad*PrecioUnitario)-MontoDescuento)*(PorcentajeImpuesto/100),2))
		--select @EINVOICETYPE,((cantidad*PrecioUnitario)-MontoDescuento)*(PorcentajeImpuesto/100),CONVERT(DECIMAL(15,2),round(((cantidad*PrecioUnitario)-MontoDescuento)*(PorcentajeImpuesto/100),2)),MontoImpuesto
		 --from #DetalleXML
	--COMMIT TRAN
	-- FIN NODO DETALLE	
		
----------------------------
----PARAMETROS NODO DE REFERENCIA (Devolución)
--------------------------
	CREATE TABLE #ReferenciaXML( 
		TpoDocRef				VARCHAR(2),
		NumeracionReferencia	VARCHAR(50), -- Tiquete electrónico o clave numerica
		CodigoReferencia		VARCHAR(2),
		TransactionId			VARCHAR(50),
		ReceiptId				VARCHAR(20))
	
	IF(@EINVOICETYPE = 3)
	BEGIN 
		DECLARE @ReturnTransactionId VARCHAR(50)
		
		SET	@ReturnTransactionId		= (SELECT TOP 1 ReturnTransactionId FROM #DetalleXML)
		
		INSERT INTO #ReferenciaXML
			SELECT TpoDocRef			= '04',
				NumeracionReferencia	= EINV.EINVOICEID, --@ReturnTransactionId,
				CodigoReferencia		= '99',--CASE WHEN @NUMBEROFITEMS = TRANS.NUMBEROFITEMS THEN '01' ELSE '03' END, -- 1 anula referencia , 2 corrige texto, 3 corrige monto
				TRANS.TransactionId, 
				TRANS.ReceiptId 
			FROM RETAILTRANSACTIONTABLE TRANS (NOLOCK)	
			INNER JOIN EINVOICELOG EINV
				ON EINV.INVOICEID		= TRANS.TransactionId
				AND EINV.STORE			= CONVERT(INT,TRANS.STORE)
				AND EINV.EINVOICEID IS NOT NULL
			WHERE TRANS.DATAAREAID		= @DATAAREAID
				AND TRANS.TRANSACTIONID = @ReturnTransactionId			
	END
	-- FIN NODO REFERENCIA		
----------------------------
----PARAMETROS NODO DE TOTALES
--------------------------
	DECLARE @TotalServGravados			DECIMAL(15,2)
	DECLARE	@TotalServExentos			DECIMAL(15,2)
	DECLARE	@TotalMercanciasGravadas	DECIMAL(15,2)
	DECLARE	@TotalMercanciasExentas		DECIMAL(15,2)
	DECLARE	@TotalGravado				DECIMAL(15,2)
	DECLARE	@TotalExento				DECIMAL(15,2)
	DECLARE	@TotalVenta					DECIMAL(15,2)
	DECLARE	@TotalDescuentos			DECIMAL(15,2)
	DECLARE	@TotalVentaNeta				DECIMAL(15,2)
	DECLARE	@TotalImpuesto				DECIMAL(15,2)
	DECLARE	@TotalComprobante			DECIMAL(15,2)
	--DECLARE @NumeroFactura			VARCHAR(20)

	CREATE TABLE #TotalesXML( 
		TotalServGravados				DECIMAL(15,2),
		TotalServExentos				DECIMAL(15,2),
		TotalServExonerados				DECIMAL(15,2),  --VERSION 4.3
		TotalMercanciasGravadas			DECIMAL(15,2),
		TotalMercanciasExentas			DECIMAL(15,2),
		TotalMercanciasExoneradas		DECIMAL(15,2),  --VERSION 4.3
		TotalGravado					DECIMAL(15,2),
		TotalExento						DECIMAL(15,2),
		TotalExonerado					DECIMAL (15,2),  --VERSION 4.3
		TotalOtrosCargos				DECIMAL (15,2),  --VERSION 4.3
		TotalIVADevuelto				DECIMAL (15,2), --VERSION 4.3
		TotalVenta						DECIMAL(15,2),
		TotalDescuentos					DECIMAL(15,2),
		TotalVentaNeta					DECIMAL(15,2),
		TotalImpuesto					DECIMAL(15,2),
		TotalComprobante				DECIMAL(15,2),
		TransactionId					VARCHAR(50),
		ReceiptId						VARCHAR(20))
	
	SELECT 
		@TotalServGravados				= ISNULL(SUM(F.TotalServGravados),0),
		@TotalServExentos				= ISNULL(SUM(F.TotalServExentos),0),
		@TotalMercanciasGravadas		= ISNULL(SUM(F.TotalMercanciasGravadas),0),
		@TotalMercanciasExentas			= ISNULL(SUM(F.TotalMercanciasExentas),0),
		@TotalGravado					= @TotalServGravados + @TotalMercanciasGravadas,  
		@TotalExento					= @TotalServExentos + @TotalMercanciasExentas, 
		@TotalVenta						= @TotalGravado + @TotalExento,
		@TotalDescuentos				= SUM(F.TotalDescuentos),
		@TotalVentaNeta					= @TotalVenta - @TotalDescuentos,
		@TotalImpuesto					= SUM(F.TotalImpuesto),
		@TotalComprobante				= @TotalVentaNeta + @TotalImpuesto
	FROM (
		SELECT 
			TotalServGravados			= CASE WHEN Tipo  = 'S' AND TaxItemGroup <> 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalServExentos			= CASE WHEN Tipo  = 'S' AND TaxItemGroup = 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalMercanciasGravadas		= CASE WHEN Tipo  = 'M' AND TaxItemGroup <> 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalMercanciasExentas		= CASE WHEN Tipo  = 'M' AND TaxItemGroup = 'EXENTOS' THEN SUM(PrecioBruto) END,
			TotalDescuentos				= SUM(MontoDescuento),
			TotalImpuesto				= SUM(MontoImpuesto),
			TransactionId, ReceiptId
		FROM #DetalleXML
		GROUP BY Tipo, TransactionId, ReceiptId, TaxItemGroup ) F
	GROUP BY F.TransactionId, F.ReceiptId

	--BEGIN TRAN 
	--Nodo Totales
	INSERT INTO #TotalesXML
		SELECT 
			@TotalServGravados,
			@TotalServExentos,
			0,  --VERSION 4.3
			@TotalMercanciasGravadas,
			@TotalMercanciasExentas,
			0,  --VERSION 4.3
			@TotalGravado,  
			@TotalExento,
			0,  --VERSION 4.3
			0,	--VERSION 4.3
			0,	--VERSION 4.3
			@TotalVenta	,
			@TotalDescuentos,
			@TotalVentaNeta,
			@TotalImpuesto,
			@TotalComprobante,
			@TRANSACTION,
			@RECEIPT
			
	-- FIN NODO TOTALES	
	--COMMIT TRAN
	--CREACION XML	
	DECLARE	@XMLString	VARCHAR(MAX)	
	 
	SET @XMLString =  (	
		SELECT 
			(SELECT 
				TransactionId,
				FechaFactura,	
				(SELECT NumCuenta FOR	XML PATH('Emisor'), TYPE),
				CodigoActividad, --Version  4.3
				TipoCambio,
				TipoDoc,			
				CondicionVenta,
				Moneda,
				idMedioPago,		
				Sucursal,
				Terminal,	
				SituacionEnvio,
				( SELECT NombreReceptor, TipoIdentificacion, IdentificacionReceptor, CorreoElectronicoReceptor, CopiaCortesia FOR XML PATH('Receptor'), TYPE)
			FROM #EncabezadoXML
			FOR	XML PATH('Encabezado'),TYPE),
			(SELECT
				(SELECT 
				CodigoProducto,
				Cantidad,
				UnidadMedida,
				DetalleMerc,
				PrecioUnitario,
				case when MontoDescuento<>0 then (SELECT MontoDescuento,NaturalezaDescuento FOR	XML PATH('Descuentos'), TYPE)else '' end, --Version 4.3
				(SELECT CodigoImpuesto,CodigoTarifa,PorcentajeImpuesto, MontoImpuesto FOR	XML PATH('Impuesto'), TYPE, ROOT ('Impuestos'))	--Version 4.3 CodigoTarifa		
				FROM #DetalleXML
				FOR	XML PATH('Linea'),TYPE)
			FOR XML PATH('Detalle'),TYPE),		
				(SELECT TpoDocRef = ISNULL(TpoDocRef,0),	NumeracionReferencia = ISNULL(NumeracionReferencia,0), CodigoReferencia = ISNULL(CodigoReferencia,0)
					FROM #ReferenciaXML
				FOR	XML PATH('Referencia'), TYPE, ROOT ('InformacionDeReferencia')),
			(SELECT 
				TotalServGravados,
				TotalServExentos,
				TotalServExonerados,   --VERSION 4.3
				TotalMercanciasGravadas,
				TotalMercanciasExentas,
				TotalMercanciasExoneradas,  --VERSION 4.3
				TotalGravado,  
				TotalExento,
				TotalExonerado,  --VERSION 4.3
				TotalOtrosCargos,
				TotalIVADevuelto,
				TotalVenta	,
				TotalDescuentos,
				TotalVentaNeta,
				TotalImpuesto,
				TotalComprobante
			FROM #TotalesXML
			FOR	XML PATH('Totales'),TYPE)
		FOR XML PATH('FacturaElectronicaXML'), ROOT('root')
		)

	DECLARE	@XMLPrefix	VARCHAR(100)	
	SET @XMLPrefix = '<?xml version="1.0" encoding="UTF-8"?>'
	
	SET @XMLString = @XMLPrefix + @XMLString

	DECLARE @INVOICE_EXIST NVARCHAR(20)

	SELECT @INVOICE_EXIST =  INVOICEID
	FROM EINVOICELOG
	WHERE INVOICEID = @TRANSACTION
		AND EINVOICEID <> ''

	IF(@INVOICE_EXIST IS NULL)
	BEGIN

		--SELECT XMLstr = @XMLString, Email = @EMAIL, Pwd = @PASS, Factura = @TRANSACTION, Sucursal = @SUC, Term = @TERM
		EXEC dbo.InsertarDocumentosEInvoice @XMLString, @EMAIL, @PASS, @TRANSACTION, @SUC, @TERM
	
		--Inserta el XML en una tabla temporal para recorrerlo
		CREATE TABLE #XMLwithOpenXML (
			Id				INT IDENTITY PRIMARY KEY,
			XMLData			XML,
			LoadedDateTime  DATETIME )

		--WAITFOR DELAY '00:00:02' 

		DECLARE @RetailStore	Varchar(10);
		DECLARE @XmlRespPath	Varchar(100)		= 'C:\\FacturaElectronica\Homex\DocumentosInsertados\'
		DECLARE @FileName		Varchar(100);
		DECLARE @Ext			Varchar(6)			= '.xml';
		DECLARE @Sufix			Varchar(11)			= '_respuesta'
		DECLARE @sql			nvarchar(MAX);

		SET @RetailStore	= '_' +  @SUC +'_' + @TERM	; 

		SET @FileName		= @XmlRespPath  + @TRANSACTION +  @RetailStore + @Sufix + @ExT;
		
		Select @sql			= N'INSERT INTO #XMLwithOpenXML(XMLData, LoadedDateTime)
					SELECT CONVERT(XML, BulkColumn) AS BulkColumn, GETDATE() 
				FROM OPENROWSET(BULK ''' + @FileName + ''', SINGLE_BLOB) AS x;'
	
		EXEC sp_executesql @sql
	
		DECLARE @XML AS XML, @hDoc AS INT --, @SQL NVARCHAR (MAX)
		DECLARE @EINVOICEID		NVARCHAR(30)
		DECLARE @SEQUENCENUM	NVARCHAR(50)
		DECLARE @ERRORCODE		NVARCHAR(20)
		DECLARE @ERRORMSG		NVARCHAR(255)
		DECLARE @XMLRECEIVEDSTR NVARCHAR(MAX)

		SELECT @XML = XMLData, @XMLRECEIVEDSTR = convert(varchar(max),XMLData )
		FROM #XMLwithOpenXML	
	
		EXEC sp_xml_preparedocument @hDoc OUTPUT, @XML

		SELECT 
			@SEQUENCENUM	= ClaveNumerica ,
			@EINVOICEID		= NumConsecutivoCompr,
			@ERRORCODE		= CodigoError,
			@ERRORMSG		= DescripcionError
		FROM OPENXML(@hDoc, 'root/FacturaElectronicaXML')
		WITH 
		(	
			ClaveNumerica [nvarchar](50)  'ClaveNumerica',
			NumConsecutivoCompr [nvarchar](30) 'NumConsecutivoCompr',
			CodigoError [varchar](20) 'CodigoError',		
			DescripcionError [varchar](255) 'DescripcionError'	
		)

		EXEC sp_xml_removedocument @hDoc
	 	
		--BEGIN TRY	
		--BEGIN TRAN
		--SELECT @RECEIPTID
		
		--
		IF exists(SELECT INVOICEID FROM EINVOICELOG WHERE INVOICEID = @TRANSACTION AND (EINVOICEID = '' or einvoiceid is null))
			BEGIN
			UPDATE EINVOICELOG
				SET EINVOICEID			= @EINVOICEID, 
					EINVOICESEQUENCEKEY = @SEQUENCENUM, 
					ERRORMSG			= @ERRORMSG,
					XMLSENDED			= @XMLString, 
					XMLRECEIVED			= @XMLRECEIVEDSTR,
					ERRORCODE			= @ERRORCODE, 
					POSCREATEDDATE		= @POSDATETIME

				WHERE INVOICEID = @TRANSACTION;
		END
		ELSE
			BEGIN
				INSERT INTO EINVOICELOG (DATAAREAID, EINVOICEID, EINVOICESEQUENCEKEY, ERRORMSG, XMLSENDED, XMLRECEIVED, INVOICEID, ERRORCODE, GI_SENTSITUATION, STORE, TERMINAL, GI_EINVOICEINSERTSTATUS, EINVOICETYPE, CREATEDDATETIME, RECEIPTID, POSCREATEDDATE)
								VALUES (@DATAAREAID, ISNULL(@EINVOICEID, ''), ISNULL(@SEQUENCENUM,''), @ERRORMSG, @XMLString,@XMLRECEIVEDSTR, @TRANSACTION, @ERRORCODE, 1, @SUC, @TERM, 0, @EINVOICETYPE,DATEADD(HOUR,-6,GETUTCDATE()), @RECEIPT,  @POSDATETIME)
			END
		
		
		--COMMIT TRAN
	END
	
	DROP TABLE #RETTRANSTMP
	DROP TABLE #EncabezadoXML
	DROP TABLE #DetalleXML
	DROP TABLE #TotalesXML
	DROP TABLE #XMLwithOpenXML
END

```

### 12. Crear el Store Procedure (SP) Check_Insert_EInvoicelog <a name="doce"></a>
Este SP se encarga de verificar periódicamente que todas las transacciones que se encuentran en la tabla de _RetailTransactionTable_ también se encuentre en la tabla de logs _EInvoiceLog_.

Para crear este SP ejecute el siguiente script:

```SQL
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:          Jeffrey Camareno Fonseca
-- Create date:     2018-08-21
-- Description:	    Verifica lo las facturas que faltan en EInvoiceLog vs Tabla de transacciones del POS
-- =============================================
-- EXEC dbo.Check_Insert_EInvoicelog N'fmcm'
CREATE PROCEDURE [dbo].[Check_Insert_EInvoicelog]
	@DATAAREAID VARCHAR(5)	
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	SET NOCOUNT ON;

	CREATE TABLE #TEMP (
		 ID INT,
		 DATAAREAID VARCHAR(50),
		 RECEIPTID	VARCHAR(50),
		 TRANSDATE	DATE,
		 CREATEDDATETIME DATETIME,
		 TRANSACTIONID VARCHAR(50),
		 ISRETURN	INT
		 --CheckDate	DATE 
		 )

	DECLARE @ROWNCOUNT INT	
	DECLARE @Counter INT = 1 	
	DECLARE	@TransactionId  VARCHAR(20) 

	INSERT INTO #TEMP 
		--SELECT ID = ROW_NUMBER() OVER(ORDER BY CREATEDDATETIME ASC), F.*  
		--FROM (
		SELECT ID = ROW_NUMBER() OVER(ORDER BY TRANS.CREATEDDATETIME ASC), PAYM.DATAAREAID, PAYM.RECEIPTID, PAYM.TRANSDATE, CREATEDDATETIME = DATEADD(HOUR,-6,TRANS.CREATEDDATETIME), PAYM.TRANSACTIONID,
			ISRETURN = CASE WHEN TRANS.SALEISRETURNSALE = 1 OR TRANS.PAYMENTAMOUNT < 0 THEN 1 ELSE 0 END
		FROM [ax].[RETAILTRANSACTIONPAYMENTTRANS]	PAYM  (NOLOCK)
		INNER JOIN [ax].[RETAILTRANSACTIONTABLE]	TRANS (NOLOCK)
			ON TRANS.DATAAREAID = PAYM.DATAAREAID
			AND TRANS.TYPE = 2
			AND TRANS.TRANSACTIONID = PAYM.TRANSACTIONID
		LEFT JOIN [dbo].[EINVOICELOG]	EINV			   (NOLOCK)
			ON EINV.INVOICEID   = PAYM.TRANSACTIONID 
			AND EINV.DATAAREAID = PAYM.DATAAREAID
		WHERE PAYM.DATAAREAID = @DATAAREAID
			AND TRANS.TRANSDATE  >= CONVERT(date, GETDATE()-24) --revisa 3 dias hacia atras hasta el dia actual
			AND (EINV.EINVOICEID = '' OR EINV.EINVOICEID IS NULL)
			--AND RETRIES < 4 --# DE INTENTOS MAXIMOS PARA REENVIAR FACTURAS ELECTRONICAS
			--AND PAYM.TRANSACTIONID IN ('0003-000015-41755','0003-005-98613')
		GROUP BY PAYM.DATAAREAID, PAYM.RECEIPTID, PAYM.TRANSDATE, TRANS.CREATEDDATETIME, PAYM.TRANSACTIONID, TRANS.SALEISRETURNSALE, TRANS.PAYMENTAMOUNT-- , CONVERT(date, GETDATE())
		ORDER BY TRANS.CREATEDDATETIME ASC
			--) AS F
			--WHERE ISRETURN = 0
	
	SET @ROWNCOUNT = @@ROWCOUNT;
	
	WHILE @Counter <= @ROWNCOUNT
	BEGIN
		SELECT @TransactionId = T.TRANSACTIONID
		FROM #TEMP T
		WHERE t.ID = @Counter
					 
		--SELECT @Counter, @TransactionId;
		EXEC DBO.Insert_EInvoice @DATAAREAID, @TransactionId;		
		SET @Counter = @Counter + 1
 
	END
	
	SELECT * FROM #TEMP
	
	DROP TABLE #TEMP
END

```

### 13. Crear Job EInvoiceFMCM <a name="trece"></a>
Este _Job_ se encargará de ejecutar cada 10 minutos (por defecto) el SP InsertEInvoice

Para crear el trigger, siga los siguientes pasos:

    |-- SQL Server Agent
        |-- New job
            |-- General
                |-- Name: EInvoiceFMCM
                |-- Description: Consume el servicio web de GTI para insertar las ventas en hacienda y crear el tiquete electrónico.
             |-- Steps
                |-- New
                    |-- General
                        |-- Step name: InsertarDocumentos
                        |-- Type: Transact-SQL script (T-SQL)
                        |-- Run as: -none-
                        |-- Database: --seleccionar base de datos transaccional de retail-- (RetailChannelDB / RetailStore / etc)
                        |-- Command> EXEC dbo.Check_Insert_EInvoicelog N'fmcm'
                    |-- Advanced
                        |-- On success action: Quit the job reporting success
                        |-- On failure action: Quit the job reporting failure
             |-- Schedules
                 |-- New
                     |-- Name: InsertarDocumentos
                     |-- Schedule type: Recurring
                     |-- Occurs: Daily
                     |-- Recurs every: 1 day
                     |-- Occurs every: 10 minutes
                     |-- Starting at: 8:00 am
                     |-- Ending at: 8:00 pm
                     |-- Check: No end date

Una vez creado el _Job_ únicamente deberá verificar si se encuentra activado.
