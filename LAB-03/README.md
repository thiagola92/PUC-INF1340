# Tabelas

## Bases de dados do sistema de vendas

```SQL
CREATE TABLE NotasVenda(
    Numero          INTEGER,
    DataEmissao     DATE,
    FormaPagamento  VARCHAR(50),
    CodigoCliente   INTEGER,
    CPFVendedor     VARCHAR(50),
    
    PRIMARY
        KEY(Numero),
    FOREIGN				-- NEW
        KEY(CPFVendedor)		-- NEW
        REFERENCES Funcionario(CPF),	-- NEW
    FOREIGN
        KEY(CodigoCliente)
        REFERENCES Cliente(Codigo)
);
```

```SQL
CREATE TABLE ItensNota(
    Numero              INTEGER,
    NumeroMercadoria    INTEGER,
    Quantidade          INTEGER,
    ValorUnitario       NUMERIC(10,2),
    
    PRIMARY
        KEY(Numero, NumeroMercadoria),
    FOREIGN				-- NEW
        KEY(Numero)			-- NEW
        REFERENCES NotasVenda(Numero),	-- NEW
    FOREIGN
        KEY(NumeroMercadoria)
        REFERENCES Mercadorias(NumeroMercadoria)
);
```

```SQL
CREATE TABLE Mercadorias(
    NumeroMercadoria        INTEGER,
    Descricao               VARCHAR(255),
    QuantidadeEstoque       INTEGER	CHECK(QuantidadeEstoque >= 0),  -- NEW CONSTRAINT
    QuantidadeMin           INTEGER,					-- NEW
    QuantidadeMax           INTEGER,					-- NEW
    
    PRIMARY
        KEY(NumeroMercadoria)
);
```

```SQL
CREATE TABLE Cliente(
    Codigo              INTEGER,
    Nome                VARCHAR(255),
    Telefone            VARCHAR(255),
    Logradouro          VARCHAR(255),
    Numero              INTEGER,
    Complemento         VARCHAR(255),
    Cidade              VARCHAR(255),
    Estado              VARCHAR(2),
    NumeroContribuinte  INTEGER,
    
    PRIMARY
        KEY(Codigo)
);
```

## Base de dados do apoio ao controle de funcionários

```SQL
CREATE TABLE Funcionario(
    CPF                 VARCHAR(15),
    Nome                VARCHAR(255),
    Telefone            VARCHAR(255),
    Logradouro          VARCHAR(255),
    Numero              INTEGER,
    Complemento         VARCHAR(255),
    Cidade              VARCHAR(255),
    Estado              VARCHAR(2),
    CodigoDepartamento  INTEGER,
    
    PRIMARY
        KEY(CPF),
    FOREIGN
        KEY(CodigoDepartamento)
        REFERENCES Departamento(CodigoDepartamento)
        
);
```

```SQL
CREATE TABLE CargosFunc(
    CPF                 VARCHAR(15),
    CodigoCargo         INTEGER,
    DataInicio          DATE,
    DataFim             DATE,
    
    PRIMARY
        KEY(CPF, CodigoCargo, DataInicio),
    FOREIGN
        KEY(CPF)
        REFERENCES Funcionario(CPF),
    FOREIGN
        KEY(CodigoCargo)
        REFERENCES Cargo(Codigo)
);
```

```SQL
CREATE TABLE Departamento(
    CodigoDepartamento  INTEGER,
    Nome                VARCHAR(255),
    CPFChefe            VARCHAR(15),
    
    PRIMARY
        KEY(CodigoDepartamento)
);
```

```SQL
CREATE TABLE Cargo(
    Codigo      INTEGER,
    Descricao   VARCHAR(255),
    SalarioBase NUMERIC(15,2),
    
    PRIMARY
        KEY(Codigo)
);
```

## Alteraçõoes nas bases de dados anteriores

```SQL
ALTER TABLE NotasVenda
ADD
	CONSTRAINT cpfVendedorLigadoAoFuncionario
	FOREIGN
		KEY(CPFVendedor)
		REFERENCES Funcionario(CPF);
```

```SQL
ALTER TABLE ItensNota
ADD
	CONSTRAINT ItemVendidoLigadoNotaVenda
	FOREIGN
		KEY(Numero)
		REFERENCES NotasVenda(Numero)
```

# 1

```SQL
CREATE TABLE NotaFiscalCompra(
    Numero                  INTEGER,
    DataCompra              DATE,
    CodigoFornecedor        INTEGER,
    
    PRIMARY
        KEY(Numero),
    FOREIGN
        KEY(CodigoFornecedor)
        REFERENCES Cliente(Codigo)
);
```

```SQL
CREATE TABLE ProdutosComprados(
    Numero              INTEGER,
    NumeroMercadoria    INTEGER,
    Quantidade          INTEGER,
    ValorUnitario       NUMERIC(10,2),
    ValorTotal          NUMERIC,
    
    PRIMARY
        KEY(Numero, NumeroMercadoria),
    FOREIGN
        KEY(Numero)
        REFERENCES NotaFiscalCompra(Numero),
    FOREIGN
        KEY(NumeroMercadoria)
        REFERENCES Mercadorias(NumeroMercadoria)
);
```

Tabela Cliente vai ser usada para armazenar também os Fornecedores.  

```SQL
CREATE VIEW Fornecedor
AS
	SELECT Cliente.*
	FROM Cliente, Notafiscalcompra
	WHERE Codigo = Codigofornecedor;
```

# 2

Uma função para pegar tudo vendido para a empresa.  
Uma função para pegar tudo comprado da empresa.  
As funções vão retornar cursor.  

```SQL
CREATE FUNCTION VendidoParaEmpresa(codigo INTEGER) RETURNS REFCURSOR AS $$
DECLARE
	cursorTabela REFCURSOR;
BEGIN
	OPEN cursorTabela FOR
	SELECT *
		FROM NotasVenda
		WHERE (codigo = CodigoCliente);
	RETURN cursorTabela;
END;
$$ LANGUAGE PLPGSQL;
```

```SQL
CREATE FUNCTION CompradoDaEmpresa(codigo INTEGER) RETURNS REFCURSOR AS $$
DECLARE
	cursorTabela REFCURSOR;
BEGIN
	OPEN cursorTabela FOR
	SELECT *
		FROM NotaFiscalCompra
		WHERE (codigo = CodigoFornecedor);
	RETURN cursorTabela;
END;
$$ LANGUAGE PLPGSQL;
```

# 3

Cade Mercadoria tem um mínimo

```SQL
ALTER TABLE Mercadorias
ADD COLUMN
	QuantidadeMin		INTEGER;
```

Placeholder.

```SQL
CREATE FUNCTION EnviarNotificacaoPorEmail() 
RETURNS TRIGGER
AS $$
	BEGIN
	RETURN NULL;
	END;
$$ LANGUAGE PLPGSQL;
```

Gatilho para avisar se descer abaixo do mínimo.

```SQL
CREATE TRIGGER ProdutoQuantidadeAbaixoDoMinimo
AFTER
	UPDATE
	ON Mercadorias
	FOR EACH ROW
	WHEN (NEW.QuantidadeEstoque <= OLD.QuantidadeMin)
EXECUTE PROCEDURE EnviarNotificacaoPorEmail();
```

# 4

Uma view para auxiliar conseguir a data de venda de cada produto.  

```SQL
CREATE VIEW DataQueProdutoFoiComprado AS
	SELECT NumeroMercadoria, DataCompra
	FROM ProdutosComprados, NotaFiscalCompra
	WHERE ProdutosComprados.Numero = NotaFiscalCompra.Numero;
```

Função para obter o menor valor possível para um produto.  

```SQL
CREATE FUNCTION PrecoDeVendaMin(codigoProduto INTEGER) RETURNS NUMERIC AS $$
DECLARE
	dataDaUltimaVenda		DATE;
	codigoDaUltimaVenda		INTEGER;
	valorDaUltimaVenda		NUMERIC;
	
BEGIN
	SELECT MAX(DataCompra)
	INTO dataDaUltimaVenda
	FROM DataQueProdutoFoiComprado
	WHERE codigoProduto = NumeroMercadoria;
		
	SELECT Numero
	INTO codigoDaUltimaVenda
	FROM NotaFiscalCompra
	WHERE dataDaUltimaVenda = DataCompra;
	
	SELECT ValorUnitario
	INTO valorDaUltimaVenda
	FROM ProdutosComprados
	WHERE (codigoDaUltimaVenda = Numero
		  AND codigoProduto = NumeroMercadoria);
		
	RETURN valorMin;
END;
$$ LANGUAGE PLPGSQL;
```

Uma função para levantar error.  

```SQL
CREATE FUNCTION PrecoAbaixoDoPermitido() 
RETURNS TRIGGER
AS $$
	BEGIN
	RAISE EXCEPTION 'Preco abaixo do permitido';
	RETURN NULL;
	END;
$$ LANGUAGE PLPGSQL;
```

Gatilho para bloquear a inserção de preços abaixos do mínimo.  

```SQL
CREATE TRIGGER PrecoDeVendaMenorQueMin
BEFORE
	INSERT
	ON ProdutosComprados
	FOR EACH ROW
	WHEN (NEW.ValorUnitario < PrecoDeVendaMin(NEW.NumeroMercadoria))
EXECUTE PROCEDURE PrecoAbaixoDoPermitido();
```

# 5

```SQL
ALTER TABLE Mercadorias
	ADD CONSTRAINT MercadoriaAcimaDeZero
		CHECK(QuantidadeEstoque >= 0)
```

# 6

Precisamos definir o máximo daquela mercadoria

```SQL
ALTER TABLE Mercadorias
ADD COLUMN
	QuantidadeMax		INTEGER;
```

```SQL
CREATE FUNCTION QuantidadeAcimaDoPermitido()
RETURNS TRIGGER
AS $$
	BEGIN
	RAISE EXCEPTION 'Quantidade acima do permitido';
	RETURN NULL;
	END;
$$ LANGUAGE PLPGSQL;
```

```SQL
CREATE TRIGGER QuantidadeDeProdutosAcimaDoMax
BEFORE
	INSERT
	ON Mercadorias
	FOR EACH ROW
	WHEN (NEW.QuantidadeEstoque > NEW.QuantidadeMax)
EXECUTE PROCEDURE QuantidadeAcimaDoPermitido();
```

# 7

```SQL
ALTER TABLE NotasVenda
ADD
	CONSTRAINT cpfVendedorLigadoAoFuncionario
	FOREIGN
		KEY(CPFVendedor)
		REFERENCES Funcionario(CPF);
```

# 8


# 9

## Cadastro de produtos com todas as suas informações

```SQL
CREATE PROCEDURE CadastroDeProduto(NumeroMercadoria INTEGER, Descricao VARCHAR,
				QuantidadeEstoque INTEGER, QuantidadeMin INTEGER,
				QuantidadeMax INTEGER)
AS $$
BEGIN
	INSERT
	INTO Mercadorias(NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QuantidadeMax)
	VALUES (NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QuantidadeMax);
END;
$$ LANGUAGE PLPGSQL;
```

## Consulta dos N maiores clientes

É preciso calcular o valor total de uma compra.  

```SQL
CREATE VIEW ValorTotalDaCompra
AS
	SELECT Numero, SUM(ValorUnitario * Quantidade) AS Total
	FROM ItensNota
	GROUP BY Numero;
```

É preciso calcular o valor total de todas as compras dos clientes.  

```SQL
CREATE VIEW ValorTotalDeTodasComprasDeUmCliente
AS
	SELECT CodigoCliente, SUM(Total) AS Total
	FROM NotasVenda, ValorTotalDaCompra
	WHERE NotasVenda.Numero = ValorTotalDaCompra.Numero
	GROUP BY CodigoCliente;
```

Uma função que retorna um cursor com os N maiores clientes.  

```SQL
CREATE FUNCTION ConsultaMaioresClientes(quantidade INTEGER) RETURNS REFCURSOR
AS $$
	DECLARE
		cursorTabela REFCURSOR;
	BEGIN
		OPEN cursorTabela FOR
		SELECT *
			FROM ValorTotalDeTodasComprasDeUmCliente
			ORDER BY Total ASC
			LIMIT quantidade;
		RETURN cursorTabela;
	END;
$$ LANGUAGE PLPGSQL;
```

## Consulta a dados de um fornecedor/cliente X

```SQL
CREATE FUNCTION ConsultaCliente(CodigoCliente INTEGER) RETURNS REFCURSOR
AS $$
	DECLARE
		cursorTabela REFCURSOR;
	BEGIN
		OPEN cursorTabela FOR
		SELECT *
			FROM Cliente
			WHERE (Codigo = CodigoCliente);
		RETURN cursorTabela;
	END;
$$ LANGUAGE PLPGSQL;
```

## Consulta a estoque e preço de um produto

```SQL
CREATE FUNCTION ConsultaEstoquePreco(CodigoProduto INTEGER, OUT ProdutoEstoque INTEGER, OUT ProdutoValor INTEGER)
AS $$
	BEGIN
		SELECT QuantidadeEstoque
		INTO ProdutoEstoque
		FROM Mercadorias
		WHERE CodigoProduto = NumeroMercadoria;
		
		SELECT FIRST(ValorUnitario)
		INTO ProdutoValor
		FROM ItensNota
		WHERE CodigoProduto = NumeroMercadoria;
	END;
$$ LANGUAGE PLPGSQL;
```

## Consulta a dados de um pedido X

Toda compra feita pela empresa é armazenada na NotaFiscalCompra, então seus pedidos estão lá.  

```SQL
CREATE FUNCTION ConsultaDadosDePedido(codigoPedido INTEGER) RETURNS REFCURSOR
AS $$
	DECLARE
		cursorTabela REFCURSOR;
	BEGIN
		OPEN cursorTabela FOR
		SELECT *
			FROM NotaFiscalCompra
			WHERE (Numero = codigoPedido);
		RETURN cursorTabela;
	END;
$$ LANGUAGE PLPGSQL;
```
