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

