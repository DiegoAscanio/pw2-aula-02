<meta charset="utf-8">

<style scoped>

body {
        font-family: Arial, sans-serif;
        margin: 2.5% 10% 0 10%;
        text-align: justify;
}

iframe {
    display: block;
    margin-left: auto;
    margin-right: auto;
    max-width: 100%;
}

img {
    display: block;
    margin-left: auto;
    margin-right: auto;
    width: 75%;
}

/* Figure / caption */
figure.youtube {
  margin: 1.5rem 0;
  text-align: center;
}

figure.youtube figcaption {
  margin-top: 0.6rem;
  font-size: 0.95rem;
  color: #333;
  line-height: 1.4;
}

/* título do vídeo */
figure.youtube .video-title {
  font-weight: 700;
  margin-bottom: 0.3rem;
}

/* metadados (canal, data) */
figure.youtube .meta {
  font-style: italic;
  font-size: 0.92rem;
  margin-bottom: 0.45rem;
}

/* citações */
figure.youtube .citation {
  font-family: Georgia, "Times New Roman", serif;
  font-size: 0.9rem;
  background: #fbfbfb;
  border-left: 3px solid #ddd;
  padding: 0.6rem 0.8rem;
  display: inline-block;
  text-align: left;
  max-width: 720px;
}

/* small helper for source link */
figure.youtube a.source {
  color: #0066cc;
  text-decoration: none;
}
figure.youtube a.source:hover { text-decoration: underline; }

</style>

<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.11.1/styles/default.min.css">
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.11.1/highlight.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.11.1/languages/java.min.js"></script>

# Aula — Persistência em SQL com **JDBC padrão** (agnóstico ao SGBD)

## DECOMDV — Programação Web II • 2025/2

**Prof. M.Sc. Diego Ascânio Santos**

## Objetivos

Ao final desta aula o estudante será capaz de:

1. Explicar **JDBC** e seu papel na comunicação com **qualquer** SGBD relacional.
2. Adicionar **manual** (pasta `lib/`) o **driver JDBC** do SGBD escolhido e configurar o *classpath*.
3. **Abrir** e **fechar** conexões JDBC explicitamente (sem *try-with-resources*), entendendo por que **fechar recursos** é crítico.
4. Usar **Statements / PreparedStatements**, **Queries** e **ResultSets** com segurança (prevenindo **SQL Injection**).
5. Implementar **Create** e **Retrieve** no repositório **Refeição** persistido via JDBC.
6. (Exercício) Implementar **Update** e **Delete**.

---

## 1. JDBC — visão geral (agnóstico ao SGBD)

* **JDBC (Java Database Connectivity)** é a API padrão do Java para falar com **bancos relacionais**.
* Você programa contra **interfaces genéricas**:
  `Connection` (conexão), `Statement/PreparedStatement` (comandos SQL), `ResultSet` (linhas retornadas).
* O que muda de um SGBD para outro é **o driver (JAR)** e **a URL JDBC**:

  * MariaDB/MySQL: `jdbc:mariadb://host:3306/banco` / `jdbc:mysql://host:3306/banco`
  * PostgreSQL: `jdbc:postgresql://host:5432/banco`
  * SQLite: `jdbc:sqlite:/caminho/arquivo.db` (sem usuário/senha)
  * SQL Server: `jdbc:sqlserver://host:1433;databaseName=banco`

---

## 2. Inclusão do driver JDBC (JAR) manualmente

1. Crie a pasta `lib/` na raiz do projeto.
2. Baixe o JAR do driver do seu SGBD (ex.: [`mariadb-java-client-2.7.2.jar`](https://downloads.mariadb.com/Connectors/java/connector-java-2.7.2/mariadb-java-client-2.7.2.jar), `postgresql-<versão>.jar`, `sqlite-jdbc-<versão>.jar`).
3. Coloque o JAR em `lib/`.
4. No VS Code (aba **Java Projects** → **Referenced Libraries**), adicione `lib/**/*.jar` ao *classpath*.

---

## 3. Conexão JDBC — **abrir** e **fechar** explicitamente (sem *try-with-resources*)

**Padrão do projeto (legado-friendly):**

* A classe **Repositório**:

  * possui **métodos** `openConnection()` e `closeConnection()`;
  * **chama `openConnection()` no construtor**;
  * **NÃO** usa *try-with-resources*; fecha **manualmente** `ResultSet`, `Statement/PreparedStatement` e, ao final, a `Connection`.

> **Por que fechar?** Conexões são recursos limitados. Deixar abertas → vazamentos, esgotamento de *pool*, travamentos e erros intermitentes.

---

## 4. Statements, Queries e ResultSets

* **Statement**: executa SQL “cru” (evite concatenar dados do usuário).
* **PreparedStatement**: SQL **parametrizado** com `?` (**recomendado** e mais seguro).
* **Queries**: `SELECT ...` → retornam **ResultSet** (cursor).
* **SQL Injection**: **nunca** montar SQL com concatenação de strings vindas do usuário.
  **Mitigue** com `PreparedStatement` + validação (tipo, tamanho, formato) + *whitelists* para nomes de tabela/colunas quando inevitável.
* **ResultSet**: use `rs.next()` e leia por nome/índice de coluna; **sempre** fechar manualmente.

---

## 5. Modelo, Tabela e Interface de Repositório

### 5.1 Modelo

```java
// src/modelos/Refeicao.java
package modelos;
import java.time.LocalDate;

public class Refeicao {
    private final String idCartaoEstudante;
    private final LocalDate dataRefeicao;

    public Refeicao(String id, LocalDate data) {
        this.idCartaoEstudante = id;
        this.dataRefeicao = data;
    }
    public String getIdCartaoEstudante() { return idCartaoEstudante; }
    public LocalDate getDataRefeicao()   { return dataRefeicao; }
}
```

### 5.2 DDL (ANSI-SQL simples)

```sql
DROP DATABASE IF EXISTS refeicoes;
CREATE DATABASE refeicoes;
USE refeicoes;

CREATE TABLE IF NOT EXISTS refeicao (
  id_cartao     VARCHAR(64) NOT NULL,
  data_refeicao DATE        NOT NULL,
  PRIMARY KEY (id_cartao, data_refeicao)
);
```

### 5.3 Interface (CRUD parcial nesta aula)

```java
// src/repositorios/RepositorioRefeicao.java
package repositorios;
import modelos.Refeicao;
import java.time.LocalDate;
import java.util.List;

public interface RepositorioRefeicao {
    void create(Refeicao r);                                   // Create
    Refeicao retrieve(String id, LocalDate data);    // Read (por chave composta)
    List<Refeicao> retrieveAll();                              // Read (todos)
    // Update e Delete: exercício
    void update(Refeicao r);                                   // Update
    void delete(String id, LocalDate data);                    // Delete
}
```

---

## 6. Repositório JDBC (padrão do projeto: abrir no construtor, fechar em método)

> **Sem *try-with-resources***. Fechamento **manual** em `finally`.

```java
// src/repositorios/RepositorioRefeicaoSQL.java
package repositorios;

import modelos.Refeicao;
import java.sql.*;
import java.time.LocalDate;
import java.util.*;

public class RepositorioRefeicaoSQL implements RepositorioRefeicao {
    private final String url;
    private final String user;
    private final String pass;
    private Connection con; // mantida enquanto o repositório estiver em uso

    // Construtor chama o processo explícito de abertura
    public RepositorioRefeicaoSQL(String url, String user, String pass) {
        this.url = url; this.user = user; this.pass = pass;
        openConnection(); // padrão do projeto
    }

    // 6.1 Iniciar conexão (explícito)
    public final void openConnection() {
        if (this.con != null) return; // já aberta
        try {
            if (user == null && pass == null) {
                this.con = DriverManager.getConnection(url);     // ex.: SQLite
            } else {
                this.con = DriverManager.getConnection(url, user, pass);
            }
            // this.con.setAutoCommit(true); // default
        } catch (SQLException e) {
            throw new RuntimeException("Falha ao abrir conexão JDBC", e);
        }
    }

    // 6.2 Fechar conexão (explícito) — liberar recursos
    public final void closeConnection() {
        if (this.con != null) {
            try { if (!this.con.isClosed()) this.con.close(); }
            catch (SQLException ignored) {}
            finally { this.con = null; }
        }
    }

    @Override
    public void create(Refeicao r) {
        PreparedStatement ps = null;
        try {
            String sql = "INSERT INTO refeicao (id_cartao, data_refeicao) VALUES (?, ?)";
            ps = con.prepareStatement(sql);
            ps.setString(1, r.getIdCartaoEstudante());
            ps.setDate(2, Date.valueOf(r.getDataRefeicao()));
            ps.executeUpdate();
        } catch (SQLIntegrityConstraintViolationException dup) {
            throw new IllegalArgumentException("Refeição já registrada para este estudante nesta data.", dup);
        } catch (SQLException e) {
            throw new RuntimeException("Erro ao inserir refeição", e);
        } finally {
            if (ps != null) { try { ps.close(); } catch (SQLException ignored) {} }
        }
    }

    @Override
    public Refeicao retrieve(String idCartao, LocalDate data) {
        ensureOpen();
        PreparedStatement ps = null;
        ResultSet rs = null;
        try {
            String sql = "SELECT id_cartao, data_refeicao FROM refeicao WHERE id_cartao=? AND data_refeicao=?";
            ps = con.prepareStatement(sql);
            ps.setString(1, idCartao);
            ps.setDate(2, java.sql.Date.valueOf(data));
            rs = ps.executeQuery();
            if (rs.next()) {
                return new Refeicao(
                    rs.getString("id_cartao"),
                    rs.getDate("data_refeicao").toLocalDate()
                );
            }
            return null; // <<-- agora retorna null quando não encontra
        } catch (SQLException e) {
            throw new RuntimeException("Erro ao recuperar refeição", e);
        } finally {
            if (rs != null) { try { rs.close(); } catch (SQLException ignored) {} }
            if (ps != null) { try { ps.close(); } catch (SQLException ignored) {} }
        }
    }

    @Override
    public List<Refeicao> retrieveAll() {
        PreparedStatement ps = null;
        ResultSet rs = null;
        List<Refeicao> out = new ArrayList<>();
        try {
            String sql = "SELECT id_cartao, data_refeicao FROM refeicao ORDER BY data_refeicao DESC, id_cartao";
            ps = con.prepareStatement(sql);
            rs = ps.executeQuery();
            while (rs.next()) {
                out.add(new Refeicao(
                    rs.getString("id_cartao"),
                    rs.getDate("data_refeicao").toLocalDate()
                ));
            }
            return out;
        } catch (SQLException e) {
            throw new RuntimeException("Erro ao listar refeições", e);
        } finally {
            if (rs != null) { try { rs.close(); } catch (SQLException ignored) {} }
            if (ps != null) { try { ps.close(); } catch (SQLException ignored) {} }
        }
    }

    @Override
    public void update(Refeicao r) {
        // Exercício para os alunos
        throw new UnsupportedOperationException("Método não implementado");
    }

    @Override
    public void delete(String id, LocalDate data) {
        // Exercício para os alunos
        throw new UnsupportedOperationException("Método não implementado");
    }
}
```

### 6.3 Uso básico (inicializar, usar, **fechar**)

```java
// src/Main.java
import repositorios.RepositorioRefeicaoSQL;
import modelos.Refeicao;
import java.time.LocalDate;

public class Main {
    public static void main(String[] args) {
        // Ajuste a URL ao seu SGBD
        String url  = "jdbc:mariadb://localhost:3306/refeicoes";
        String user = "aluno";
        String pass = "123456";

        RepositorioRefeicaoSQL repo = new RepositorioRefeicaoSQL(url, user, pass);
        try {
            repo.create(new Refeicao("CARTAO123", LocalDate.of(2025, 9, 1)));
            System.out.println(
                Refeicao refeicao = repo.retrieve("CARTAO123", LocalDate.of(2025, 9, 1));
            );
        } finally {
            // Padrão do projeto: fechamento explícito
            repo.closeConnection();
        }
    }
}
```

---

## 7. Exercício (para os alunos): **Update** e **Delete**

Implemente **no mesmo padrão** as operações **Update** e **Delete** na classe `RepositorioRefeicaoSQL`, seguindo a interface `RepositorioRefeicao`.

### 7.3 Checklist de entrega

* Usa **PreparedStatement** (sem concatenação de strings).
* Fecha **ResultSet** / **Statement** **manualmente** (em `finally`).
* Trata **duplicidade** (chave primária) e **não encontrado** (linhas afetadas = 0).
* Pequeno **programa de teste**: *insere → consulta → atualiza → consulta → deleta → consulta*.
* **Fecha a conexão** ao final (`closeConnection()`).

---

### Encerramento

* **Padrão adotado para todo o projeto**: `openConnection()` chamado no **construtor** e `closeConnection()` explícito ao terminar — **sem** *try-with-resources* — para enfatizar **ciclo de vida dos recursos** e preparar o aluno para **códigos legados**.
* **Segurança**: parâmetros (`?`) em **PreparedStatement** e validação de entradas → previnem **SQL Injection**.
* **Agnóstico ao SGBD**: troque apenas **driver** e **URL**; o repositório permanece.

<script>hljs.highlightAll();</script>
