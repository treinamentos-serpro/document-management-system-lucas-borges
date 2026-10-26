# Especificação - Document Management System

> Especificação funcional e técnica inicial para orientar o desenvolvimento do sistema.
> Esta versão define o escopo e os contratos; não autoriza nem descreve a execução de alterações nos arquivos de backend ou frontend.

## 1. Objetivo

Disponibilizar uma aplicação web para que usuários enviem, consultem e baixem seus documentos, mantendo os arquivos no filesystem local e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Envio de um arquivo por requisição.
- Listagem dos documentos associados ao usuário da requisição.
- Download de um documento pelo identificador, respeitando a propriedade do documento.
- Identificação simples do usuário para separar documentos por dono, sem autenticação ou gestão de contas nesta fase.
- Armazenamento de arquivos no diretório local `backend/storage` e de metadados em memória.
- Interface web para envio, listagem e download, consumindo a API pelo prefixo `/api` configurado no proxy do Vite.

### Fora do escopo

- Armazenamento em nuvem, banco de dados ou qualquer persistência externa.
- Versionamento, edição, exclusão ou compartilhamento de documentos.
- Cadastro, autenticação, autorização baseada em papéis ou recuperação de identidade do usuário.
- Upload múltiplo em uma única requisição.
- Busca avançada, ordenação configurável ou paginação nesta fase inicial.
- Garantia de preservação de metadados após reinício do processo.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O usuário pode enviar um arquivo por `multipart/form-data`, usando o campo `file`. |
| RF-02 | A API deve rejeitar requisições sem arquivo, com mais de um arquivo ou que excedam o limite configurado, sem registrar metadados de upload incompleto. |
| RF-03 | A aplicação deve atribuir ao documento um identificador único, preservar o nome original como metadado e registrar tamanho, instante do upload e dono. |
| RF-04 | O usuário pode listar apenas os documentos associados ao identificador de usuário enviado na requisição. |
| RF-05 | O usuário pode baixar um documento pelo identificador somente se ele pertencer ao usuário da requisição. |
| RF-06 | A API deve responder com erro apropriado quando o documento não existir, não pertencer ao usuário ou não puder ser lido do armazenamento local. |
| RF-07 | A interface deve permitir selecionar e enviar um arquivo, exibir o resultado ou erro do envio, listar documentos e iniciar o download de um documento listado. |
| RF-08 | A interface deve apresentar estados de carregamento, lista vazia e falha de comunicação sem perder a possibilidade de tentar novamente. |

### Identificação do usuário

Nesta fase, o cliente informa o identificador por meio do cabeçalho `X-User-Id`. O valor é obrigatório, não vazio e tratado como identificador opaco. Esse mecanismo serve somente à separação funcional dos documentos; não comprova identidade e não deve ser considerado uma barreira de segurança. Autenticação real permanece fora do escopo.

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos devem ser gravados exclusivamente no filesystem local, em `backend/storage`, usando `multer` com `diskStorage`. |
| RNF-02 | Os metadados devem permanecer em memória nesta fase; uma reinicialização pode remover os metadados mesmo que os arquivos continuem no disco. |
| RNF-03 | O diretório de armazenamento e o limite máximo de upload devem ser configuráveis por variáveis de ambiente, com valores padrão documentados. |
| RNF-04 | O nome físico do arquivo deve ser gerado pela aplicação e não derivado diretamente do nome fornecido pelo cliente; o nome original é apenas metadado e nome sugerido no download. |
| RNF-05 | As rotas devem validar entradas e tratar erros de leitura e gravação sem expor caminhos locais, stack traces ou detalhes internos ao cliente. |
| RNF-06 | A API deve retornar JSON UTF-8 nos resultados e erros, exceto no corpo binário de download. Datas devem usar ISO 8601 em UTC. |
| RNF-07 | O backend deve seguir a Clean Architecture simples: `routes -> controllers -> services -> repositories`; dependências apontam para dentro e regras de negócio ficam nos serviços. |
| RNF-08 | O frontend deve usar componentes React e acessar o backend via `fetch` usando o prefixo `/api`; configurações de ambiente não devem ser codificadas como segredos no cliente. |
| RNF-09 | A solução deve evitar dependências de serviços externos de armazenamento e manter compatibilidade com o runner nativo `node:test` já adotado para testes de backend. |

### Configuração

- `STORAGE_DIR`: diretório físico dos uploads; padrão `backend/storage` relativo à raiz do backend.
- `MAX_FILE_SIZE_BYTES`: tamanho máximo permitido em bytes; padrão inicial recomendado de 10 MiB (`10485760`).
- A aplicação deve falhar com erro claro na inicialização se o diretório não puder ser preparado ou utilizado. Valores inválidos para configuração devem ser rejeitados na inicialização.

O limite padrão é uma decisão inicial configurável e pode ser ajustado antes da implementação sem mudar o contrato da API.

## 5. Modelo de dados (metadados do documento)

### Metadados expostos pela API

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | sim | Identificador único, gerado pela aplicação (UUID). |
| `originalName` | string | sim | Nome original informado pelo cliente, preservado como metadado. |
| `size` | number | sim | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | sim | Instante de recebimento do upload em ISO 8601 UTC. |
| `owner` | string | sim | Identificador recebido no cabeçalho `X-User-Id`. |

### Metadados internos

O repositório pode manter, além dos campos expostos, o nome físico gerado para localizar o arquivo. Esse valor é interno, não deve ser retornado ao cliente e não deve ser construído a partir de caminhos fornecidos pelo cliente. O conteúdo binário reside no disco, não no objeto de metadados.

### Regras de consistência

- O `id` é único durante a vida do processo.
- `size` corresponde ao tamanho efetivamente gravado.
- `uploadedAt` é definido pelo servidor após o arquivo ser recebido com sucesso.
- Um registro só é incluído na coleção em memória após a gravação bem-sucedida.
- Se a gravação falhar, nenhum registro deve ser criado; arquivos temporários/incompletos devem ser removidos quando possível.
- Listagens não incluem o caminho físico nem metadados internos.

## 6. Contratos de API

### Convenções gerais

- Rotas do backend: `/upload`, `/documents` e `/documents/:id/download`.
- A aplicação frontend acessa essas rotas pelo prefixo `/api` via proxy do Vite, por exemplo `/api/documents`.
- As requisições de upload e listagem devem incluir `X-User-Id`.
- O download também requer `X-User-Id`; o cliente web pode obter o binário via `fetch` e iniciar o download localmente.
- Respostas de erro seguem o formato `{ "error": { "code": "...", "message": "..." } }`.

### POST /upload

- Cabeçalho obrigatório: `X-User-Id: <identificador>`.
- Entrada: `multipart/form-data` com exatamente um arquivo no campo `file`.
- Regras: arquivo obrigatório; limite máximo conforme `MAX_FILE_SIZE_BYTES`; nome físico gerado pelo servidor. Não há restrição de extensão ou tipo MIME nesta versão; o MIME informado pelo cliente não é considerado prova do conteúdo.
- Sucesso: `201 Created`, `Content-Type: application/json`.

```json
{
  "id": "8e039148-f433-4f3f-9d5d-5a9448f6ebc8",
  "originalName": "relatorio.pdf",
  "size": 24576,
  "uploadedAt": "2026-10-26T14:30:00.000Z",
  "owner": "usuario-123"
}
```

- Erros previstos: `400 Bad Request` para arquivo ausente/inválido; `413 Payload Too Large` para arquivo acima do limite; `400 Bad Request` para cabeçalho de usuário ausente ou vazio; `500 Internal Server Error` para falha inesperada de armazenamento.

### GET /documents

- Cabeçalho obrigatório: `X-User-Id: <identificador>`.
- Entrada: sem corpo.
- Sucesso: `200 OK`, `Content-Type: application/json`; retorna uma lista, inclusive vazia, contendo somente documentos do usuário.

```json
[
  {
    "id": "8e039148-f433-4f3f-9d5d-5a9448f6ebc8",
    "originalName": "relatorio.pdf",
    "size": 24576,
    "uploadedAt": "2026-10-26T14:30:00.000Z",
    "owner": "usuario-123"
  }
]
```

- Erros previstos: `400 Bad Request` para cabeçalho de usuário ausente ou vazio; `500 Internal Server Error` para falha inesperada.

### GET /documents/:id/download

- Cabeçalho obrigatório: `X-User-Id: <identificador>`.
- Entrada: identificador do documento no parâmetro `id`; sem corpo.
- Sucesso: `200 OK`, conteúdo binário do arquivo, `Content-Disposition: attachment` com nome original sanitizado e `Content-Length` quando disponível.
- Erros previstos: `400 Bad Request` para cabeçalho ou identificador inválido; `404 Not Found` para documento inexistente ou pertencente a outro usuário (mesma resposta para evitar revelar a existência de documentos alheios); `500 Internal Server Error` para falha de leitura. O caminho físico nunca é aceito como entrada da rota.

### Semântica dos erros

| Status | Código sugerido | Uso |
| --- | --- | --- |
| 400 | `INVALID_REQUEST` | Campo ou cabeçalho obrigatório ausente/inválido. |
| 404 | `DOCUMENT_NOT_FOUND` | Documento inexistente ou não pertencente ao usuário. |
| 413 | `FILE_TOO_LARGE` | Limite de upload excedido. |
| 500 | `INTERNAL_ERROR` | Falha inesperada; mensagem não expõe detalhes internos. |

## 7. Decisões arquiteturais

- Backend em Node.js + Express, CommonJS; frontend em React + Vite, ESM.
- Fluxo do backend: `routes -> controllers -> services -> repositories`.
- `routes` declaram métodos e caminhos e delegam aos controllers.
- `controllers` traduzem HTTP para chamadas de serviço, validam entrada básica e formatam respostas.
- `services` aplicam regras de upload, propriedade, listagem e download sem depender de Express.
- `repositories` abstraem o armazenamento local e a coleção em memória de metadados.
- `multer` com `diskStorage` recebe o upload e grava no filesystem local, usando nome físico gerado pelo servidor.
- Arquivos ficam em `backend/storage` por padrão; metadados ficam em memória. Não há banco de dados ou armazenamento remoto.
- A interface utiliza `fetch` e o proxy `/api` do Vite; não acessa diretamente o filesystem.
- A identificação por `X-User-Id` é provisória e não substitui autenticação. Nenhum mecanismo de autenticação deve ser presumido pelos contratos.

## 8. Plano de execução

Este plano cobre exclusivamente a etapa de especificação e preparação. A execução de arquivos de backend e frontend fica explicitamente para uma etapa futura, fora deste plano.

1. Consolidar o objetivo, os limites do escopo e as exclusões funcionais.
2. Validar as decisões pendentes do produto: identificação provisória por `X-User-Id`, limite padrão de 10 MiB e ausência de filtro por tipo de arquivo.
3. Revisar os requisitos funcionais e não funcionais quanto à testabilidade e compatibilidade com a arquitetura definida.
4. Aprovar os campos de metadados, as regras de propriedade e a distinção entre dados expostos e internos.
5. Validar os contratos HTTP, códigos de status, formato de erro e comportamento de upload, listagem e download.
6. Definir os critérios de aceite da futura implementação a partir dos requisitos e contratos desta especificação.
7. Revisar e aprovar esta especificação como referência antes de iniciar qualquer execução em arquivos de backend ou frontend.

### Critérios de aceite da especificação

- As operações de envio, listagem e download têm entradas, respostas e erros definidos.
- A propriedade por usuário e os limites da identificação provisória estão explícitos.
- Está definido que arquivos usam somente armazenamento local via `multer` com `diskStorage` e metadados em memória.
- A arquitetura e os limites entre camadas estão registrados.
- O plano não contém execução nem alteração de arquivos de backend ou frontend.
