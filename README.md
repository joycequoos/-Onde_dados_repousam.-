# Onde os Dados Repousam

[← Voltar a Tuning SQL Server](https://github.com/joycequoos/SQL_Server_Developer_Tuning_Codigoscom_maximo_desempenho./blob/main/README.md)

Introdução à página de dados do SQL Server: os conceitos fundamentais de armazenamento (Páginas de Dados e Extents) e de uso de memória pela instância (Buffer Pool, Min/Max Server Memory), essenciais para começar a trabalhar com performance.

## Índice

- [Páginas de Dados](#páginas-de-dados)
- [Extent (Extensão)](#extent-extensão)
- [Memória da Instância do SQL Server](#memória-da-instância-do-sql-server)
  - [Buffer Pool (ou Buffer Cache)](#buffer-pool-ou-buffer-cache)
  - [Configurando a Memória no SQL Server](#configurando-a-memória-no-sql-server)
  - [Min Server Memory](#min-server-memory)
  - [Max Server Memory](#max-server-memory)
- [Scripts e Referências](#scripts-e-referências)
- [Próximos Passos](#próximos-passos)

---

## Páginas de Dados

Todos os dados enviados das aplicações e sistemas para um banco de dados são gravados em tabelas, através de instruções `INSERT` e `UPDATE` (para manutenção) ou `DELETE` (para exclusão).

Internamente, as tabelas possuem outra definição, chamada de **objeto de alocação de dados**. Em cada arquivo de dados existem áreas pré-definidas onde os dados são gravados — essas áreas são associadas aos objetos de alocação e é nelas que os dados são gravados em formato de registro. Essas áreas são conhecidas como **Páginas de Dados**.

| Característica | Descrição |
| --- | --- |
| **Unidade fundamental** | A página de dados é a menor alocação utilizada pelo SQL Server, sendo a unidade fundamental de armazenamento |
| **Tamanho fixo** | Cada página de dados tem exatamente **8 KB (8192 bytes)**, divididos entre cabeçalho, área de dados e slot de controle |
| **Exclusividade** | Uma página de dados é exclusiva de um único objeto de alocação, mas um objeto de alocação pode ter diversas páginas de dados |
| **Capacidade útil** | Em cada linha, só é possível armazenar até **8060 bytes** de dados dentro de uma página |

Para verificar o espaço ocupado por uma tabela:

```sql
EXECUTE sp_spaceused 'NomeDaTabela'
```

> 👇 **Para saber mais:** documentação oficial de [`sp_spaceused`](https://docs.microsoft.com/pt-br/sql/relational-databases/system-stored-procedures/sp-spaceused-transact-sql).

## Extent (Extensão)

Um **Extent** é um agrupamento lógico de páginas de dados, cujo objetivo é gerenciar melhor o espaço alocado. Um Extent tem exatamente **8 páginas de dados**, totalizando **64 KB**.

Existem dois tipos de Extent:

| Tipo | Descrição |
| --- | --- |
| **Mixed Extent** (Misto) | As páginas de dados pertencem a objetos de alocação diferentes |
| **Uniform Extent** (Uniforme) | As páginas de dados pertencem exclusivamente a um único objeto de alocação |

Uma nova tabela é alocada inicialmente em um Mixed Extent, utilizando uma única página de dados. Se a tabela precisar de uma nova página e o Extent ainda tiver páginas não utilizadas, o SQL Server continua alocando ali mesmo, junto com páginas de outros objetos de alocação. Quando não há mais páginas livres no Mixed Extent, o SQL Server passa a alocar todas as novas páginas em um Uniform Extent.

> 👇 **Saiba mais:** [`01 - Página e Extent.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/01 - Página e Extent.sql>)

## Memória da Instância do SQL Server

Uma pergunta simples ajuda a entender por que a memória é tão importante: **é mais rápido acessar os dados em memória ou em disco?** A resposta já justifica por que o SQL Server precisa de memória suficiente para atender à carga de dados.

- **Quanto mais memória, melhor.** Ela é usada para carregar, do disco, os dados que precisam ser acessados, mantendo-os em uma área do SQL Server conhecida como **Buffer Pool**.
- Servidores com **16 GB, 32 GB ou 64 GB** atendem à maioria das demandas — mas há instalações com mais de **512 GB** de memória.
- A recomendação mínima da Microsoft é de apenas **1 GB**, mas o recomendado é começar com pelo menos **4 GB**, sempre validando com uma análise real do ambiente para o dimensionamento correto.

### Buffer Pool (ou Buffer Cache)

Um **buffer** é uma área de **8 KB** na memória, onde o SQL Server armazena as páginas de dados lidas dos objetos de alocação em disco — tarefa de responsabilidade do **Gerenciador de Buffer**.

- O dado permanece no buffer até que o Gerenciador de Buffer precise da área para carregar novas páginas. Os buffers mais antigos e com dados modificados são gravados em disco e liberados para novas páginas.
- Quando o SQL Server precisa de um dado e ele já está no buffer, ocorre uma **leitura lógica**. Se o dado não estiver no buffer, o SQL Server realiza uma **leitura física** do disco para o Buffer Pool.
- A área de memória do Buffer Pool é controlada por duas configurações da instância: **Min Server Memory** e **Max Server Memory**.

### Configurando a Memória no SQL Server

Ao instalar o SQL Server, ele configura automaticamente o uso da memória disponível no servidor, através das opções **Max Server Memory** e **Min Server Memory**. Para consultá-las:

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE WITH OVERRIDE

EXECUTE sp_configure 'min server memory (MB)'
EXECUTE sp_configure 'max server memory (MB)'
```

Por padrão, o resultado normalmente mostra **1024 KB** de memória mínima e **2147483647 KB** (aproximadamente 2 TB) de memória máxima — o valor máximo teórico suportado, não um valor real de uso.

### Min Server Memory

- **Não** representa a memória mínima que o SQL Server efetivamente utiliza.
- Ao inicializar, o serviço aloca inicialmente **128 KB** e aguarda as atividades de inclusão, alteração e exclusão de dados pela aplicação. Conforme as consultas são executadas, o SQL Server carrega dados do disco para a memória já reservada.
- Enquanto a alocação não ultrapassa o valor definido em `Min Server Memory`, essa memória pertence ao SQL Server e **não é devolvida** ao sistema operacional, mesmo que ele solicite.
- Após ultrapassar esse valor mínimo, o SQL Server continua alocando mais memória normalmente — mas, se o SO precisar dessa memória de volta, o SQL Server pode liberá-la, até o limite mínimo configurado.

### Max Server Memory

- **Não** representa a memória máxima que o SQL Server efetivamente utiliza no dia a dia.
- O SQL Server continua alocando dados do disco para a memória até atingir o valor definido em `Max Server Memory`. Ao atingir esse limite, se precisar alocar novos dados, ele grava em disco os dados mais antigos, libera a área de memória correspondente e então aloca os novos dados.
- Se o sistema operacional (ou outras aplicações) precisar de memória e não houver memória livre suficiente, o SO pode solicitá-la ao SQL Server. Caso a memória reservada pelo SQL Server não esteja em uso, ele grava os dados pendentes em disco e libera essa memória para o SO — até o limite definido em `Min Server Memory`.

> 📺 **Referência:** [vídeo sobre o assunto no YouTube](https://www.youtube.com/watch?v=OijdLj4lw5c).
>
> 👇 **Saiba mais:** [`02 - Arquitetura da Memória.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/02 - Arquitetura da Memoria.sql>)

## Scripts e Referências

| Script | Conteúdo |
| --- | --- |
| [`01 - Página e Extent.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/01 - Página e Extent.sql>) | Exemplos práticos sobre Páginas de Dados e Extents |
| [`02 - Arquitetura da Memoria.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/02 - Arquitetura da Memoria.sql>) | Exemplos práticos sobre a arquitetura de memória do SQL Server |

## Próximos Passos

- Testar `sp_spaceused` em tabelas reais do ambiente e comparar o espaço reservado com o espaço realmente utilizado pelos dados.
- Explorar a DMV `sys.dm_os_buffer_descriptors` para visualizar, na prática, quais páginas estão atualmente no Buffer Pool.
- Documentar um caso real de ajuste de `Min Server Memory`/`Max Server Memory` em um ambiente com múltiplas instâncias compartilhando o mesmo servidor.
- Conectar este conteúdo ao próximo módulo do curso de Tuning, sobre design de banco de dados e estrutura de índices (B-Tree). 
