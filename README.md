# migracao-bancos
Migra dados entre determinadas tabelas de um banco e outro banco

Ao criar uma nova base de dados em uma empresa, deparei-me com a seguinte situação: várias tabelas do novo banco de dados teriam as mesmas configurações de um banco antigo, mais especificamente referente à configurações padrão.
Não poderia fazer um restore porque seriam somente algumas tabelas e que para dificultar teriam que manter o mesmo ID para compatibilidade com a aplicação.

Desta forma, criar algumas funções e um script para ler tabelas que atendessem determinados critérios de nome: config, configuração, configurações e suas variações, apagasse os dados da tabela destino e inserisse os dados a partir da origem.
O código ficou muito rápido e pode ser facilmente adaptado para ler outras tabelas

````
/* 
Copia dados de "FaculdadeOriginal" para "Faculdade"
facilitando o trabalho de inserção dos dados
*/

use Faculdade

-- Cria funções para a migração
-- Função que retorna chave primária das tabelas
CREATE OR ALTER FUNCTION fn_chavePrimaria(@Tabela VARCHAR(250))
RETURNS VARCHAR(500)
AS
BEGIN
    DECLARE @Result VARCHAR(500);

    SELECT @Result = ISNULL(
		STUFF((
        SELECT ', ' + name
        FROM SYS.COLUMNS
        WHERE object_name(object_id) = @Tabela
		and is_identity = 1
        ORDER BY column_id
        FOR XML PATH(''), TYPE).value('.', 'VARCHAR(MAX)'), 1, 2, ''),'');

    RETURN @Result;
END;
GO

-- Função que retorna colunas de cada tabela
CREATE OR ALTER FUNCTION fn_colunas(@Tabela VARCHAR(250))
RETURNS VARCHAR(5000)
AS
BEGIN
    DECLARE @Result VARCHAR(5000);

    SELECT @Result = ISNULL(
		STUFF((
        SELECT ', ' + name
        FROM SYS.COLUMNS
        WHERE object_name(object_id) = @TABELA
        ORDER BY column_id
        FOR XML PATH(''), TYPE).value('.', 'VARCHAR(MAX)'), 1, 2, ''),'');
	
	RETURN @Result;
END

-- Função que retorna colunas que não são IDENTITY de cada tabela
CREATE OR ALTER FUNCTION fn_colunas_semidentity(@Tabela VARCHAR(250))
RETURNS VARCHAR(5000)
AS
BEGIN
    DECLARE @Result VARCHAR(5000);

    SELECT @Result = ISNULL(
		STUFF((
        SELECT ', ' + name
        FROM SYS.COLUMNS
        WHERE object_name(object_id) = @TABELA
		and is_identity = 0
        ORDER BY column_id
        FOR XML PATH(''), TYPE).value('.', 'VARCHAR(MAX)'), 1, 2, ''),'');
	
	RETURN @Result;
END

-- Lê a estrutura das tabelas de config*.*, configuracao*.* ou configuracoes*.*
-- na Faculdade que NÃO POSSUAM identity
DECLARE @Tabela NVARCHAR(MAX)
DECLARE @SQL NVARCHAR(MAX)
DECLARE @SQL_SEMIDENTITY NVARCHAR(MAX)

DECLARE cursor_tabelas CURSOR FOR
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
AND (TABLE_NAME LIKE '%config%'
     OR TABLE_NAME LIKE '%configuracao%'
     OR TABLE_NAME LIKE '%configuracoes%')
	and dbo.fn_chavePrimaria(table_name) <> '' -- Não tem IDENTITY
OPEN cursor_tabelas

FETCH NEXT FROM cursor_tabelas INTO @Tabela

-- Gera scripts de tabelas que NÃO TEM ID
WHILE @@FETCH_STATUS = 0
BEGIN
	PRINT 'Importando tabela ' + @Tabela
    SET @SQL =  ' TRUNCATE TABLE Faculdade.dbo.' + @tabela + ';' +
				' INSERT INTO Faculdade.dbo.' + @Tabela + ' SELECT * FROM faculdadeOriginal.dbo.' + @Tabela 

    SET @SQL = ' SET IDENTITY_INSERT ' + @TABELA + ' ON ' +
	           @SQL_SEMIDENTITY +
			   ' SET IDENTITY_INSERT ' + @TABELA + ' OFF '
	BEGIN TRY
		EXEC sp_executesql @SQL
	END TRY
	BEGIN CATCH
		EXEC sp_executesql @SQL_SEMIDENTITY
	END CATCH
	EXEC sp_executesql @SQL
	PRINT @SQL

    FETCH NEXT FROM cursor_tabelas INTO @Tabela
END

CLOSE cursor_tabelas
DEALLOCATE cursor_tabelas

TRUNCATE TABLE Faculdade.dbo.ConfiguracaoIntranetDocente; INSERT INTO Faculdade.dbo.ConfiguracaoIntranetDocente SELECT * FROM faculdadeOriginal.dbo.ConfiguracaoIntranetDocente
TRUNCATE TABLE Faculdade.dbo.ConfiguracaoMatriculaBoletoGratuito; INSERT INTO Faculdade.dbo.ConfiguracaoMatriculaBoletoGratuito SELECT * FROM faculdadeOriginal.dbo.ConfiguracaoMatriculaBoletoGratuito
TRUNCATE TABLE Faculdade.dbo.ConfiguracaoPeriodoRematricula_Datas; INSERT INTO Faculdade.dbo.ConfiguracaoPeriodoRematricula_Datas SELECT * FROM faculdadeOriginal.dbo.ConfiguracaoPeriodoRematricula_Datas
TRUNCATE TABLE Faculdade.dbo.ConfiguracoesGerais; INSERT INTO Faculdade.dbo.ConfiguracoesGerais SELECT * FROM faculdadeOriginal.dbo.ConfiguracoesGerais
TRUNCATE TABLE Faculdade.dbo.ConfiguracoesGeraisAcesso; INSERT INTO Faculdade.dbo.ConfiguracoesGeraisAcesso SELECT * FROM faculdadeOriginal.dbo.ConfiguracoesGeraisAcesso
TRUNCATE TABLE Faculdade.dbo.ConfiguracaoIntranetAcademica; INSERT INTO Faculdade.dbo.ConfiguracaoIntranetAcademica SELECT * FROM faculdadeOriginal.dbo.ConfiguracaoIntranetAcademica
 
SET IDENTITY_INSERT IndiceFinanceiro ON
DELETE FROM Faculdade.dbo.IndiceFinanceiro; 
INSERT INTO Faculdade.dbo.IndiceFinanceiro (idIndiceFinanceiro,descricao)
SELECT idIndiceFinanceiro,descricao FROM faculdadeOriginal.dbo.IndiceFinanceiro
SET IDENTITY_INSERT IndiceFinanceiro OFF

-- Lê a estrutura das tabelas de config*.*, configuracao*.* ou configuracoes*.*
-- na Faculdade que possuam identity
DECLARE @Tabela NVARCHAR(MAX)
DECLARE @SQL NVARCHAR(MAX)
DECLARE @SQL_SEMIDENTITY NVARCHAR(MAX)
DECLARE @COLUNAS VARCHAR(5000)

DECLARE cursor_tabelas CURSOR FOR
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
AND (TABLE_NAME LIKE '%config%'
     OR TABLE_NAME LIKE '%configuracao%'
     OR TABLE_NAME LIKE '%configuracoes%')
	and dbo.fn_chavePrimaria(table_name) <> '' -- com IDENTITY
OPEN cursor_tabelas

FETCH NEXT FROM cursor_tabelas INTO @Tabela

-- Gera scripts de tabelas que TEM ID
-- Desabilita Identity, inclui o registro (para ficar com mesmo ID e manter a integridade relacional)
-- E depois habilita Identity
WHILE @@FETCH_STATUS = 0
BEGIN
	PRINT 'Importando tabela ' + @Tabela
	SET @COLUNAS = ' ' + dbo.fn_colunas(@Tabela)
    SET @SQL =  ' DELETE FROM Faculdade.dbo.' + @tabela + ';' +
				' SET IDENTITY_INSERT ' + @TABELA + ' ON; ' +
				' INSERT INTO Faculdade.dbo.' + @Tabela + 
				' (' + @COLUNAS + ')' +
				' SELECT ' + @COLUNAS + ' FROM faculdadeOriginal.dbo.' + @Tabela +
				' SET IDENTITY_INSERT ' + @TABELA + ' OFF; '

	print @SQL
	EXEC sp_executesql @SQL

    FETCH NEXT FROM cursor_tabelas INTO @Tabela
END

CLOSE cursor_tabelas
DEALLOCATE cursor_tabelas

-- Última configuração que não pode ser importada
-- tem que ser inserida
INSERT INTO ConfiguracoesGerais (
    PermitirControleAcessoPorCurso, 
    DadosCabecalhoRelatorio, 
    NumeroTentativasParaBloqueio, 
    TempoDeEsperaParaVerificacao, 
    QuantidadeMinimaCaracteresSenha, 
    HtmlAreaLivre, 
    IdConfiguracoesGerais, 
    CaminhoDaPastaLogomarca, 
    NomeDaImagemDaLogomarca, 
    AlturaDaLogomarca, 
    LarguraDaLogomarca, 
    EnabledMFA
) VALUES (
    0, 
    'Universidade Exemplo  CNPJ: 18.472.078/0001-50  FAC, 123 - Cidade Jardim, Crato - CE, 60515-000', 
    0, 
    0, 
    0, 
    NULL, 
    1, 
    '/logo_instituicao/', 
    'logomarca.png', 
    NULL, 
    NULL, 
    0
);
