# SqlServer
Querys e rotinas que facilitam nosso dia a dia.

# Paginando o resultado de uma query

```sh
DECLARE @PageSize As Int = 10
DECLARE @PageNumber As Int = 1
 SELECT
      Data
    , Valor
FROM Pedido
ORDER BY Data
    OFFSET @PageSize * (@PageNumber - 1) ROWS
    FETCH NEXT @PageSize ROWS ONLY;
```

# IIF - Alternativa ao CASE WHEN
Operador ternário. Simplifica o uso excessivo de CASE WHEN, que às vezes acaba tornando a consulta ilegível.

```sh
SELECT
      Data
    , Valor
    , IIF(Pago = 1, 'Sim', 'Não') As Pago
FROM Pedido

```

# LAG/LEAD 
Funções que permite acessar alguma coluna da linha anterior ou posterior, respectivamente.

```sh
-- Retorna NULL
SELECT
      Data
    , Valor
    , LAG(Data, 1) OVER(ORDER BY Data ASC) As PedidoAnteriorEm
FROM Pedido 
ORDER BY Data ASC
 
-- Retorna 2016-01-02 00:00:00.000
SELECT
      Data
    , Valor
    , LEAD(Data, 1) OVER(ORDER BY Data ASC) As ProximoPedidoEm
FROM Pedido 
ORDER BY Data ASC
```

# EXCLUSÃO DE OBJETOS 
Sintaxe mais simples para remoção de objetos.

```sh
-- Antes
IF OBJECT_ID('Pedido', 'U') IS NOT NULL
    DROP TABLE Pedido
 
-- Agora
DROP TABLE IF EXISTS Pedido
```

# STRING_SPLIT 
Função que recebe uma string (podendo ser uma coluna de uma tabela) separada por um determinado caractere e permite utilizar o resultado na cláusula FROM ou JOIN.

```sh
SELECT value FROM string_split('PAULO;AGATHA;FERNANDA', ';')
```

# Script responsável por buscar um campo específico em todas as tabelas do banco de dados

```sh
--use NomeDoBancoDeDados

--Buscar campo em todas as tabelas
declare @campo varchar(max)

SET @campo = 'DataModificacao';


SELECT 
	T.name AS Tabela, 
	C.name AS Coluna
FROM 
	sys.sysobjects    AS T (NOLOCK) 
INNER JOIN sys.all_columns AS C (NOLOCK) ON T.id = C.object_id AND T.XTYPE = 'U' 
WHERE 
	C.NAME LIKE '%' + @campo + '%'
ORDER BY 
	T.name ASC
```
