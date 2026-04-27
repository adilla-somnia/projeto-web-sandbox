# Sistema de Pesquisa de Satisfação e Avaliação Docente

## 1. Visão Geral

Este documento descreve as regras de negócio do sistema de pesquisa de satisfação institucional e avaliação docente.

O sistema permite a criação, resposta e análise de pesquisas aplicadas a contextos acadêmicos e institucionais.

---

## 2. Autenticação e Autorização

- O acesso ao sistema é realizado via autenticação JWT (Bearer Token)
- O token contém:
  - userId
  - role

### Perfis de usuário

- ADMIN
- GESTOR
- DOCENTE
- ALUNO

Cada perfil possui permissões específicas no sistema.

---

## 3. Regras de Usuário (ADMIN)

- Apenas ADMIN pode:
  - criar usuários
  - editar usuários
  - excluir usuários

### Restrições:
- Um ADMIN não pode:
  - remover a própria role de admin
  - deletar a si próprio
- O sistema deve sempre manter pelo menos 1 ADMIN ativo

---

## 4. Gestão Institucional e Acadêmica

Apenas ADMIN pode gerenciar:

### Institucional
- Campus
- Setor
- Serviço

### Acadêmico
- Curso
- Disciplina
- Turma
- Período
- Matrícula

---

## 5. Regras de Unicidade

As seguintes entidades não podem possuir duplicidade:

- Período → ``ano + semestre``
- Disciplina → ``nome + cursoId``
- Curso → ``nome + campusId + nivel``
- Setor → ``nome + campusId``
- Serviço → ``nome + setorId``
- Campus → ``nome + cidade + estado``
- Turma → ``disciplinaId + periodoId + docenteId``
- Matrícula → ``turmaId + alunoId``

---

## 6. Regras de Período

- Ano deve estar entre:
  - 2000 e (ano atual + 1)

- Semestre:
  - 1 ou 2

- Não pode haver duplicação de ano + semestre

---

## 7. Pesquisas (MongoDB)

As pesquisas são armazenadas no MongoDB e podem ser:

- SATISFACAO (vinculada a um Serviço)
- AVALIACAO (vinculada a uma Turma)

### Estrutura base:
- tipo
- tipoId
- titulo
- descricao
- startDate
- endDate
- status

### Regra de unicidade:
- titulo + tipo + tipoId + startDate + endDate

---

## 8. Ciclo de vida da Pesquisa

### Status possíveis:

- INATIVA
- PUBLICADA
- FECHADA

### Regras:

- Ao criar:
  - ``status = INATIVA``
- PUBLICADA:
  - ``startDate <= hoje <= endDate``
- FECHADA:
  - ``hoje > endDate``

### Restrições:
- startDate não pode ser no passado
- endDate é inclusivo
- após FECHADA, não aceita respostas

---

## 9. Criação de Pesquisas

### 9.1 Pesquisa de Satisfação

- Criada por GESTOR
- Vinculada a um Serviço
- Contém questões definidas manualmente
- Disponível para todos os alunos

---

### 9.2 Avaliação Docente

- Criada por GESTOR
- Vinculada a uma Turma

### Regras:
- título e descrição são gerados automaticamente
- baseada em critérios (ENUM)

Cada critério gera:
- uma questão do tipo ESCALA
- escala de 1 a 6

Inclui obrigatoriamente:
- uma questão final opcional do tipo ABERTA

---

## 10. Respostas (MongoDB)

- Apenas ALUNO pode responder
- Cada aluno responde apenas uma vez por pesquisa
- Respostas são associadas a alunoHashId
- Pesquisas fechadas não podem ser respondidas

### Regras de acesso:

- Avaliação docente:
  - aluno deve estar matriculado na turma

- Pesquisa de satisfação:
  - disponível para todos os alunos

### Restrição:
- aluno não pode visualizar suas respostas após envio

---

## 11. Matrícula e Acesso

- O aluno só pode responder avaliações de turmas nas quais está matriculado

---

## 12. Relatórios

### Gestor:
- acesso a relatórios agregados parciais e finalizados de todas as pesquisas

### Docente:
- acesso apenas aos relatórios parciais e finalizados de pesquisas onde foi o docente avaliado

---

## 13. Logs e Auditoria

- O sistema registra logs de ações realizadas por usuários
- Todas as entidades possuem:
  - createdAt
  - updatedAt

---

## 14. Notificações

### ALUNO
- nova pesquisa disponível
- lembrete 1 semana antes do fim (se não respondeu)

### DOCENTE
- resultado de avaliação disponível

### GESTOR
- resultado de pesquisa disponível


---

## 15. Exclusão de Pesquisas

- Pesquisas **não** podem ser deletadas
- Tentativas de DELETE são bloqueadas pelo sistema
- Respostas e questões não possuem soft delete

---

## 16. Integridade Histórica das Pesquisas

As pesquisas possuem caráter histórico e não podem ser invalidadas por alterações ou remoções de suas entidades de origem.

### 16.1 Regra de integridade

Uma pesquisa, após sua criação, deve manter consistência histórica independentemente de alterações nas entidades relacionadas no sistema relacional (MySQL), como:

- Campus
- Curso
- Disciplina
- Turma
- Serviço
- Setor
- Período

---

### 16.2 Problema de dependência

Como as pesquisas são armazenadas no MongoDB e referenciam entidades do MySQL, existe risco de quebra de referência caso essas entidades sejam alteradas ou removidas.

---

### 16.3 Estratégia adotada

Para garantir consistência histórica, o sistema adota uma combinação de:

#### a) Não exclusão lógica de entidades críticas
Entidades acadêmicas e institucionais não devem ser removidas fisicamente caso estejam associadas a pesquisas existentes.

Quando necessário, deve ser utilizada a estratégia de desativação, que é a `deletedAt`.

---

#### b) Snapshot de dados na criação da pesquisa

No momento da criação de uma pesquisa, o sistema parmazena uma cópia mínima dos dados relevantes da entidade relacionada no MongoDB.

Exemplo:

- nome da turma
- disciplina associada
- docente responsável
- curso associado
- campus vinculado

Esse snapshot garante que os relatórios permaneçam consistentes mesmo após alterações futuras no sistema relacional.

---

### 16.4 Regra de ouro

Pesquisas são consideradas **registros históricos imutáveis**.

Alterações em entidades base do sistema não devem impactar o conteúdo já respondido ou consolidado de uma pesquisa.