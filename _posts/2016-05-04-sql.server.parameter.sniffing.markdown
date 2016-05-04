---
layout: post
title:  "SQL Server Parameter Sniffing"
date:   2016-05-04 16:48:00 -0300
categories: sql-server sql database
---
Usar variáveis locais: http://stackoverflow.com/questions/6585417/stored-procedure-slow-when-called-from-web-fast-from-management-studio#13823047 

Limpar buffers: http://stackoverflow.com/questions/834124/ado-net-calling-t-sql-stored-procedure-causes-a-sqltimeoutexception#839055 
DBCC DROPCLEANBUFFERS
DBCC FREEPROCCACHE

Setar opção ARITHABORT
{% highlight sql %}
SET ARITHABORT ON; SELECT ...
{% endhighlight %}


________________________________________

	Applications using
ADO .Net, ODBC or OLE DB	SSMS, 
Query Analyzer	SQLCMD,
OSQL, BCP,
SQL Server Agent	ISQL, 
DB-Library
ANSI_NULL_DFLT_ON	ON	ON	ON	OFF
ANSI_NULLS	ON	ON	ON	OFF
ANSI_PADDING	ON	ON	ON	OFF
ANSI_WARNINGS	ON	ON	ON	OFF
CONACT_NULLS_YIELD_NULL	ON	ON	ON	OFF
QUOTED_IDENTIFIER	ON	ON	OFF	OFF
ARITHABORT	OFF	ON	OFF	OFF


 

________________________________________

A Non-Solution
Before I go into the real solutions, let me first point out that adding SET ARITHABORT ON to your procedure is not a solution. It will seem to work when you try it. But that is only because you recreated the procedure which forced a new compilation and then the next invocation sniffed the current set of parameters. SET ARITHABORT ON is only a placebo, and not even a good one. The problem will most likely come back. It will not even help you avoid the confusion with different performance in the application and SSMS, because the overall cache entry will still haveARITHABORT OFF as its plan attribute.


________________________________________

Executar recompilando a SP:
{% highlight c# %}
cmd.CommandType = CommandType.Text;
cmd.Text = "EXECUTE memdb_get_updated_customers @tstamp WITH RECOMPILE";
{% endhighlight %}


________________________________________

One solution to address this is to force recompilation every time with the RECOMPILE query hint:

{% highlight sql %}
CREATE PROCEDURE List_orders_12 @custid   nchar(5),
                                @fromdate datetime,
                                @todate   datetime AS
SELECT *
FROM   Orders
WHERE  CustomerID = @custid
  AND  OrderDate BETWEEN @fromdate AND @todate
OPTION (RECOMPILE)
With this hint, SQL Server will compile the query every time. Rather than using the query hint, you can tell SQL Server to recompile the entire procedure on each invocation:
CREATE PROCEDURE List_orders_12 @custid   nchar(5),
                                @fromdate datetime,
                                @todate   datetime WITH RECOMPILE AS
{% endhighlight %}

For the example at hand, it has no importance which you use, since the procedure consists of a single statement. But for a long procedure with many statements of which only one is problematic, WITH RECOMPILE is clearly an inferior choice, due to the increase in compilation overhead. An interesting effect of WITH RECOMPILE, though, is that the plan is never put into the cache at all, whereas this happens when you use OPTION (RECOMPILE).


________________________________________

Query para verificar os parâmetros "sniffeds" (em cache):

Fonte: http://www.sommarskog.se/query-plan-mysteries.html 

Outras referências:
http://stackoverflow.com/questions/2248112/query-times-out-when-executed-from-web-but-super-fast-when-executed-from-ssms
http://sqlperformance.com/2013/08/t-sql-queries/parameter-sniffing-embedding-and-the-recompile-options
