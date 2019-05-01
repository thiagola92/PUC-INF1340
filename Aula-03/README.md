### SGBD Centralizado
Todo o banco de dados está centralizado em um local.  
Vamos supor que temos os seguintes sites  
Site de São Paulo  
Site do Rio de Janeiro  
Site da Bahia  
...  
O banco de dados se localiza em Rio de Janeiro, ou seja, todos os outros sites tem que buscar informação no banco de dados do Rio de Janeiro.  

Os sites precisam estar preparados para caso não consigam se comunicar com o site do Rio de Janeiro. Por exemplo, se a rede cair os outros sites tem que estar preparados para lidar com essa situação.  

### SGBD Distribuido
Existe mais que um banco de dados, distribuido de diversas maneiras.  
Seguindo o princípio da localidade, você deixa os dados perto do local onde eles são mais usados.  

Site de São Paulo teria o banco de dados com os dados dos funcionário de São paulo  
Site do Rio de Janeiro teria o banco de dados com os dados dos funcionário do Rio de Janeiro    

Para obter os dados de todos funcionários, você precisa pegar a informação de todos os bancos de dados.  

Nem todos precisam ter banco de dados próprio.  

### SGBD Cliente/Servidor
O servidor armazena o banco de dados e os clientes fazem request de informações para o servidor.  

Camada Cliente  
Camada Servidor  

### SGBD 3 Camadas
Agora existe mais uma camada, a camada que lida com as requisições do cliente.  
Tira o trabalho do banco de dados de lidar com o cliente.  

Camada Cliente (navegador)  
Camada Aplicação (exemplo: apache)  
Camada Banco de Dados (banco de dados)    

### Modelagem Conceitual
Modelo Entidade Relacionamento (MER) ou Diagrama de Classes ...  

### Modelagem Lógica
Quando você aplica tecnologia no modelo conecitual, você obtem o modelo lógico.  

### Modelagem Física
Quando você representa tudo no banco de dados, obtemos modelo físico.  
