# SqlServer
Querys e rotinas que facilitam nosso dia a dia.

##### Paginando o resultado de uma query

```sql
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

##### IIF - Alternativa ao CASE WHEN
Operador ternário. Simplifica o uso excessivo de CASE WHEN, que às vezes acaba tornando a consulta ilegível.

```sql
SELECT
      Data
    , Valor
    , IIF(Pago = 1, 'Sim', 'Não') As Pago
FROM Pedido

```

##### LAG/LEAD 
Funções que permite acessar alguma coluna da linha anterior ou posterior, respectivamente.

```sql
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

##### EXCLUSÃO DE OBJETOS 
Sintaxe mais simples para remoção de objetos.

```sql
-- Antes
IF OBJECT_ID('Pedido', 'U') IS NOT NULL
    DROP TABLE Pedido
 
-- Agora
DROP TABLE IF EXISTS Pedido
```

##### STRING_SPLIT 
Função que recebe uma string (podendo ser uma coluna de uma tabela) separada por um determinado caractere e permite utilizar o resultado na cláusula FROM ou JOIN.

```sql
SELECT value FROM string_split('PAULO;AGATHA;FERNANDA', ';')
```

##### Script responsável por buscar um campo específico em todas as tabelas do banco de dados

```sql
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
##### Script responsável por localizar palavras em tabelas, tabela de sistema, procedures, views e funções

```sh
/*
U => Tabela Usuário
S => Tabela de sistema
P => Procedure
V => View
F => Function
*/

SELECT A.NAME, A.TYPE, B.TEXT
  FROM SYSOBJECTS  A (nolock)
  JOIN SYSCOMMENTS B (nolock) 
    ON A.ID = B.ID
WHERE B.TEXT LIKE '%tempo%'  --- Informação a ser procurada no corpo da procedure, funcao ou view
  AND A.TYPE = 'P'                     --- Tipo de objeto a ser localizado no caso procedure
 ORDER BY A.NAME
```

##### Script responsável por obter usuários ativos no banco
```sql
SELECT 
    DB_NAME(dbid) as DBName, 
    COUNT(dbid) as NumberOfConnections,
    loginame as LoginName
FROM
    sys.sysprocesses
WHERE 
    dbid > 0
GROUP BY 
    dbid, loginame
order by 2 desc 
```
##### Script responsável por obter consultas pesadas
```sql
SELECT TOP 10
total_worker_time/execution_count AS Avg_CPU_Time
    ,execution_count
    ,total_elapsed_time/execution_count as AVG_Run_Time
    ,(SELECT
          SUBSTRING(text,statement_start_offset/2,(CASE
                                                       WHEN statement_end_offset = -1 THEN LEN(CONVERT(nvarchar(max), text)) * 2 
                                                       ELSE statement_end_offset 
                                                   END -statement_start_offset)/2
                   ) FROM sys.dm_exec_sql_text(sql_handle)
     ) AS query_text,
     * 
FROM sys.dm_exec_query_stats 
ORDER BY Avg_CPU_Time DESC
--ORDER BY AVG_Run_Time DESC
--ORDER BY execution_count DESC
```

##### Como verificar e resolver a fragmentação de índices no SQL Server
```sh
	SELECT 
			dbschemas.[name] as 'Schema',
			dbtables.[name] as 'Table',
			dbindexes.[name] as 'Index',
			indexstats.avg_fragmentation_in_percent,
			indexstats.page_count
	 FROM 
			sys.dm_db_index_physical_stats (DB_ID(), NULL, NULL, NULL, NULL) AS indexstats
INNER JOIN sys.tables dbtables 
		on dbtables.[object_id] = indexstats.[object_id]
INNER JOIN sys.schemas dbschemas 
	    on dbtables.[schema_id] = dbschemas.[schema_id]
INNER JOIN sys.indexes AS dbindexes 
		ON dbindexes.[object_id] = indexstats.[object_id]
	   AND indexstats.index_id = dbindexes.index_id
WHERE indexstats.database_id = DB_ID() AND dbtables.[name] like '%Ultima%' 
ORDER BY indexstats.avg_fragmentation_in_percent desc

```
A seguinte tabela indica as recomendações da Microsoft sobre rebuild e reorganize:

 Fragmentação (em %)                     | Ação             | Query 
-----------------------------------------|:----------------:|------------------------------------------ 
avg_fragmentation_in_percent > 5 AND < 30| Reorganize Index | ALTER INDEX REORGANIZE 
avg_fragmentation_in_percent > 30	 | Rebuild Index    | ALTER INDEX REBUILD WITH (ONLINE = ON)

A recompilação de um índice pode ser executada online ou offline. A reorganização de um índice sempre é executada online. Para atingir disponibilidade semelhante à opção de reorganização, recrie índices online. Veja o exemplo:

```sql
--reorganizar um índice específico
ALTER INDEX IX_NAME
  ON dbo.Employee  
REORGANIZE ;   
GO  

-- reorganizar todos os índices de uma tabela
ALTER INDEX ALL ON dbo.Employee  
REORGANIZE ;   
GO  

-- rebuild de um índice específico
ALTER INDEX PK_Employee_BusinessEntityID ON dbo.Employee
REBUILD;

-- rebuild de todos índice de uma tabela
ALTER INDEX ALL ON dbo.Employee
REBUILD;
	

```

Lembre-se que o comando rebuild por padrão gera lock na tabela e leva bem mais tempo do que o reorganize. Por isto tenha cuidado ao realizar estas operações dentro do horário produtivo do seu software.


##### Como matar os processos do banco de dados
```sql
--CONSULTAR PROCESSOS
select DB_NAME(spid), *
FROM
    master..sysprocesses
WHERE
    --dbid = DB_ID('ViacaoMaua') -- Nome do database
     dbid > 4 -- Não eliminar sessões em databases de sistema
    AND spid <> @@SPID -- Não eliminar a sua própria sessão
    and status not in('sleeping', 'suspended')
--    order by last_batch asc
order by cpu desc
    

--MATAR PROCESSOS
DECLARE @query VARCHAR(MAX) = ''
 
SELECT
    @query = COALESCE(@query, ',') + 'KILL ' + CONVERT(VARCHAR, spid) + '; '
FROM
    master..sysprocesses
WHERE
    1=1
    and dbid = DB_ID('ViacaoSantaLuzia') -- Nome do database
    AND dbid > 4 -- Não eliminar sessões em databases de sistema
    AND spid <> @@SPID -- Não eliminar a sua própria sessão
	--and status <> 'sleeping'
 
IF (LEN(@query) > 0)
    --EXEC(@query)
    PRINT @QUERY

```
É possível matar o processo pelo Activity Monitor
 
- [Youtube](https://www.youtube.com/watch?v=rKvkcwFo89o)


##### Consultar tabelas locadas
```sql
SELECT distinct L.request_session_id AS SPID,
		DT.database_transaction_begin_time,
		datediff(MINUTE, DT.database_transaction_begin_time, getdate()) as tempo_transacao_aberta_minutos,
        DB_NAME(L.resource_database_id) AS DatabaseName,
        O.Name AS LockedObjectName,
        P.object_id AS LockedObjectId,
        L.resource_type AS LockedResource,
        L.request_mode AS LockType,
        ST.text AS SqlStatementText,
        ES.login_name AS LoginName,
        ES.host_name AS HostName,
        TST.is_user_transaction as IsUserTransaction,
        AT.name as TransactionName,
        CN.auth_scheme as AuthenticationMethod
FROM    sys.dm_tran_locks L
        JOIN sys.partitions P ON P.hobt_id = L.resource_associated_entity_id
        JOIN sys.objects O ON O.object_id = P.object_id
        JOIN sys.dm_exec_sessions ES ON ES.session_id = L.request_session_id
        JOIN sys.dm_tran_session_transactions TST ON ES.session_id = TST.session_id
        JOIN sys.dm_tran_active_transactions AT ON TST.transaction_id = AT.transaction_id
        JOIN sys.dm_exec_connections CN ON CN.session_id = ES.session_id

		JOIN sys.dm_tran_database_transactions DT ON DT.transaction_id = TST.transaction_id

        CROSS APPLY sys.dm_exec_sql_text(CN.most_recent_sql_handle) AS ST
WHERE  -- resource_database_id = db_id() and 
		DT.database_transaction_begin_time is not null
		
ORDER BY DT.database_transaction_begin_time desc


```

#####  Como verificar os spids responsáveis pelos bloqueios
```sql
select spid, blocked, hostname=left(hostname,20), program_name=left(program_name,20),
       WaitTime_Seg = convert(int,(waittime/1000))  ,open_tran, status
From master.dbo.sysprocesses 
where blocked > 0
and spid in (select blocked from Master.dbo.sysprocesses where blocked > 0)
order by spid
```
