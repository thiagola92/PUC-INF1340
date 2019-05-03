**Luis Claudio Cantanhêde Martins** - 1512946  
**Thiago Lages de Alencar** - 1721629  

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
    Salario             NUMERIC,	-- NEW
    Comissao            NUMERIC,	-- NEW
    
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
		
	RETURN valorDaUltimaVenda;
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

Armazenar na tabela de Funcionario o salário base e comissao.  

```SQL
ALTER TABLE Funcionario
ADD COLUMN
	Salario		NUMERIC;
```

```SQL
ALTER TABLE Funcionario
ADD COLUMN
	Comissao		NUMERIC;
```

Um gatilho que sempre que inserir em NotasVenda, calcula a comissão do funcionario.  

```SQL
CREATE FUNCTION AtualizarComissao() RETURNS TRIGGER
AS $$
DECLARE
	valorTotal			NUMERIC;
	comissaoAtual	        	NUMERIC;
	salarioB			NUMERIC;
	codigoCargoDoFuncionario	INTEGER;
BEGIN
	-- Atualizar a comissao depois dessa venda
	SELECT Total
	INTO valorTotal
	FROM ValorTotalDaCompra
	WHERE NEW.Numero = Numero;
	
	SELECT Comissao
	INTO comissaoAtual
	FROM Funcionario
	WHERE NEW.CPFVendedor = CPF;
	
	UPDATE Funcionario
	SET Comissao = (comissaoAtual + valorTotal * 5/100)
	WHERE NEW.CPFVendedor = CPF;
	
	-- Atualizar o salario total
	SELECT codigoCargo
	INTO codigoCargoDoFuncionario
	FROM CargosFunc
	WHERE NEW.CPFVendedor = CPF;
	
	SELECT SalarioBase
	INTO salarioB
	FROM Cargo
	WHERE Codigo = codigoCargoDoFuncionario;
	
	UPDATE Funcionario
	SET Salario = SalarioB + Comissao
	WHERE NEW.CPFVendedor = CPF;
	
	RETURN NULL;
END;
$$ LANGUAGE PLPGSQL;
	

CREATE TRIGGER AdicionarComissao
AFTER
	INSERT
	ON NotasVenda
	FOR EACH ROW
EXECUTE PROCEDURE AtualizarComissao();
```

# 9

## Cadastro de produtos com todas as suas informações

```SQL
CREATE FUNCTION CadastroDeProduto(NumeroMercadoria INTEGER,
                                Descricao VARCHAR,
                                QuantidadeEstoque INTEGER,
                                QuantidadeMin INTEGER,
                                QuantidadeMax INTEGER)
RETURNS void
AS $$
BEGIN
	INSERT
	INTO Mercadorias(NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QuantidadeMax)
	VALUES (NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QuantidadeMax);
END;
$$ LANGUAGE PLPGSQL;
```

Postgresql 11

```SQL
CREATE PROCEDURE CadastroDeProduto(NumeroMercadoria INTEGER,
                                Descricao VARCHAR,
                                QuantidadeEstoque INTEGER,
                                QuantidadeMin INTEGER,
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
CREATE FUNCTION ConsultaEstoquePreco(CodigoProduto INTEGER,
					OUT ProdutoEstoque INTEGER,
                    OUT ProdutoValor INTEGER)
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

# Utilizado nesse trabalho

## Tabelas
NotasVenda(Numero, DataEmissao, FormaPagamento, CodigoCliente, CPFVendedor)  
ItensNota(Numero, NumeroMercadoria, Quantidade, ValorUnitario)  
Mercadorias(NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QuantidadeMax)  
Cliente(Codigo, Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContriibuinte)  

Funcionario(CPF, Nome, Telefone, Numero, Complemento, Cidade, Estado, CodigoDepartamento, SalarioBase, Comissao)  
CargosFunc(CPF, CodigoCargo, DataInicio, DataFim)  
Departamento(CodigoDepartamento, Nome, CPFChefe)  
Cargo(Codigo, Descricao, SalarioBase)  

NotaFiscalCompra(Numero, DataCompra, CodigoFornecedor)  
ProdutosComprados(Numero, NumeroMercadoria, Quantidade, ValorUnitario, ValorTotal)  

## Views
Fornecedor(Codigo, Nome, Telefone, Logradouro, Numero, Complemento, Cidade, Estado, NumeroContriibuinte)  
DataQueProdutoFoiComprado(NumeroMercadoria, DataCompra)  
ValorTotalDaCompra(Numero, Total)  
ValorTotalDeTodasComprasDeUmCliente(CodigoCliente, Total)  

## Functions
VendidodParaEmpresa(codigo)  
CompradoDaEmpresa(codigo)  
PrecoDeVendaMin(codigoProduto)  
ConsultaMioresCLientes(Quantidade)  
COnsultaCliente(CodigoCliente)  
ConsultaEstoquePreco(CodigoProduto, OUT ProdutoEstoque, OUT ProdutoValor)  
ConsultaDadosDePedido(codigoPedido)  

## Procedures/Function
CadastroDeProduto(NumeroMercadoria, Descricao, QuantidadeEstoque, QuantidadeMin, QUantidadeMax)  

## Triggers
ProdutoQuantidadeAbaixoDoMinimo => EnviarNotificacaoPorEmail()  
PrecoDeVendaMenorQueMin => PrecoAbaixoDoPermitido()  
QuantidadeDeProdutosAcimaDoMax => QuantidadeAcimaDoPermitido()  
AdicionarComissao => AtualizarComissao()  

# Problemas encontrados

* PostgreSQL 9.4 não suporta Procedure
    * Procedures foram alterados para funções que retornam `void` 
* `PrecoDeVendaMin()` estava retornando variável que não existia.
    * Atualizado para retornar variável certa.
* `AtualizarComissao()` não chamava `return` no final.
    * Atualizado para chamar `return null`.

# Inserindo nas tabelas valores quaisqueres

```SQL
INSERT INTO Mercadorias
VALUES
(1,'descricao01', 5, 3, 7),
(2,'descricao02', 50, 20, 100),
(3,'descricao03', 30, 60, 300),
(3,'descricao03', 1, 60, 300),
(4,'descricao04', 5, 50, 250),
(5,'descricao05', 3, 40, 200),
(6,'descricao06', 7, 30, 150),
(7,'descricao07', 7, 20, 100),
(8,'descricao08', 0, 10, 50);

INSERT INTO Cliente
VALUES
(1,'nome01', 'telefone01', 'logradouro01', 1, 'complemento01', 'cidade01', '01', 1),
(2,'nome02', 'telefone02', 'logradouro02', 2, 'complemento02', 'cidade02', '02', 2),
(3,'nome03', 'telefone03', 'logradouro03', 3, 'complemento03', 'cidade03', '03', 3),
(4,'nome04', 'telefone04', 'logradouro04', 4, 'complemento04', 'cidade04', '04', 4),
(5,'nome05', 'telefone05', 'logradouro05', 5, 'complemento05', 'cidade05', '05', 5),
(6,'nome06', 'telefone06', 'logradouro06', 6, 'complemento06', 'cidade06', '06', 6);

INSERT INTO Departamento
VALUES
(1, 'departamento01', 'cpfchefe01'),
(2, 'departamento02', 'cpfchefe02'),
(3, 'departamento03', 'cpfchefe03'),
(4, 'departamento04', 'cpfchefe04'),
(5, 'departamento05', 'cpfchefe05');

INSERT INTO Cargo
VALUES
(1, 'cargo01', 1),
(2, 'cargo02', 2),
(3, 'cargo03', 3),
(4, 'cargo04', 4),
(5, 'cargo05', 5);

INSERT INTO Funcionario
VALUES
('cpf01','nome01', 'telefone01', 'logradouro01', 1, 'complemento01', 'cidade01', '01', 1, NULL, NULL),
('cpf02','nome02', 'telefone02', 'logradouro02', 2, 'complemento02', 'cidade02', '02', 2, 1000, 0),
('cpf03','nome03', 'telefone03', 'logradouro03', 3, 'complemento03', 'cidade03', '03', 3, 1500, 500);

INSERT INTO CargosFunc
VALUES
('cpf01', 1, '2019-01-01', NULL),
('cpf02', 2, '2019-02-02', NULL),
('cpf03', 3, '2019-03-03', '2019-03-04');

INSERT INTO NotasVenda
VALUES
(1, '2019-01-01', 'formapagamento01', 1, 'cpf01'),
(2, '2019-02-02', 'formapagamento02', 2, 'cpf02');

INSERT INTO ItensNota
VALUES
(1, 1, 3, 1),
(1, 2, 2, 5),
(1, 3, 1, 4),
(2, 1, 1, 1);

INSERT INTO NotaFiscalCompra
VALUES
(1, '2019-01-01', 1),
(2, '2019-02-02', 2);

INSERT INTO ProdutosComprados
VALUES
(1, 1, 5, 1, 5);
(2, 1, 10, 2, 20);
```

