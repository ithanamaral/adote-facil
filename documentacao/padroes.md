# Análise de Princípios SOLID - Projeto Adote Fácil

Este documento identifica a aplicação (ou ausência) de princípios SOLID no projeto "Adote Fácil", com base na análise da arquitetura de backend e frontend.

## 1. Single Responsibility Principle (SRP) - Princípio da Responsabilidade Única

Este princípio afirma que uma classe ou módulo deve ter uma única responsabilidade: "lógica ou regra de negócio".

### ✅ Aplicação Positiva (Backend)
O backend demonstra uma boa separação de responsabilidades:

**Rotas (`routes.ts`):** Apenas definem os endpoints e delegam a execução para os *controllers*. Não contêm lógica de negócios.

```typescript
router.post(
  '/users',
  createUserControllerInstance.handle.bind(createUserControllerInstance),
)
```
>**Sugestão:** No exemplo acima, há apenas a criação da rota, sem aplicação de lógica ou regra de negócios.


**Serviços (ex: `create-animal.ts`):** Focam exclusivamente na regra de negócio (ex: criar um animal). Não sabem como os dados são persistidos (delegam para o repositório) nem como a requisição HTTP foi feita.

```typescript
// backend/src/services/animal/create-animal.ts
export class CreateAnimalService {
  constructor(
    private readonly animalRepository: AnimalRepository,
    private readonly animalImageRepository: AnimalImageRepository,
  ) {}

  async execute(
    params: CreateAnimalDTO.Params,
  ): Promise<CreateAnimalDTO.Result> {
    // ... Lógica de negócio pura, sem req/res do Express
  }
}
```
>**Sugestão:** No exemplo acima, não há definição de rota apenas há a regra de negócio para criação do animal.

**DTOs:** O uso de classes/interfaces como `CreateAnimalDTO` separa a definição dos dados da lógica.

```typescript
// backend/src/services/animal/create-animal.ts
export namespace CreateAnimalDTO {
  export type Params = {
    name: string
    type: string
    // ... outros campos
  }
}
```
>**Sugestão:** No exemplo acima, não há definição de rota, ou lógica de negócio, apenas define os tipos de dados pertencentes ao objeto

### ❌ Pontos de Atenção (Frontend)
No frontend, há casos em que o princípio é frequentemente violado nos componentes, que tendem a acumular responsabilidades de UI e Lógica.
- **`AnimalCard.tsx`:** Este componente é responsável por:
    1. Renderizar a interface do card.
    2. Chamar a API (`handleConfirmAnimalAdoption`, `handleRemoveAnimal`).
    3. Gerenciar feedback ao usuário (`alert`).
    4. Gerenciar navegação (`window.location.href`).
- **Formulários (ex: `UpdateUserInfoForm.tsx`):** Misturam renderização com lógica de submissão e manipulação direta de armazenamento local (cookies/localStorage).


## 2. Open/Closed Principle (OCP) - Princípio Aberto/Fechado

Classes devem estar abertas para extensão, mas fechadas para modificação.

### ✅ Aplicação Positiva
- **Componentes UI (ex: `DefaultSelect.tsx`):** Recebem itens via `props`. Para adicionar novas opções, não é necessário modificar o código interno do componente, apenas passar uma nova lista.
- **Backend Services:** A estrutura permite estender funcionalidades (como validações extras) sem alterar drasticamente a lógica de persistência existente.

## 3. Liskov Substitution Principle (LSP) - Princípio da Substituição de Liskov

Objetos de uma superclasse devem ser substituíveis por objetos de suas subclasses sem quebrar a aplicação.

### ✅ Aplicação Implícita (Backend)
- Nos serviços (ex: `CreateAnimalService`), as dependências são injetadas via construtor (ex: `AnimalRepository`).
- Isso permite que qualquer implementação concreta de `AnimalRepository` (seja Postgres, Mongo ou um Mock em memória para testes) seja usada sem quebrar o serviço, respeitando o contrato da interface.

## 4. Interface Segregation Principle (ISP) - Princípio da Segregação de Interface

Muitas interfaces específicas são melhores do que uma interface única geral.

### ✅ Aplicação Positiva
- **Frontend Props:** Interfaces como `AnimalCardProps` definem estritamente o que o componente precisa para renderizar, evitando passar objetos gigantescos e desnecessários.
- **Backend DTOs:** O `CreateAnimalDTO.Params` define apenas os dados necessários para a criação, isolando campos de banco de dados (como `created_at`) que não devem ser expostos nesse contexto.

## 5. Dependency Inversion Principle (DIP) - Princípio da Inversão de Dependência

Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações.

### ⚠️ Aplicação Mista (Backend)
- **Onde funciona:** As classes de serviço recebem dependências via construtor, dependendo de abstrações (interfaces de repositório) e não instanciando classes concretas internamente com `new`.
- **Onde falha (Acoplamento):** Em alguns arquivos de serviço, há a exportação de uma instância concreta já montada:
  ```typescript
  export const createAnimalServiceInstance = new CreateAnimalService(
    animalRepositoryInstance,
    animalImageRepositoryInstance,
  )
  ```
  Isso cria acoplamento estático e dificulta testes unitários isolados nos controllers que importam essa instância diretamente.

> **Sugestão:** Utilizar um container de Injeção de Dependência ou um padrão Factory para resolver as dependências, evitando exportar instâncias "hardcoded".

## Resumo

- **Backend:** Alta maturidade na aplicação de SOLID, facilitando testes e manutenção.
- **Frontend:** Estrutura mais monolítica por componente; beneficiar-se-ia de uma separação maior entre View e Logic (Hooks).