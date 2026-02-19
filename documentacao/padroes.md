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
>**Observação:** No exemplo acima, há apenas a criação da rota, sem aplicação de lógica ou regra de negócios.


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
>**Observação:** No exemplo acima, não há definição de rota apenas há a regra de negócio para criação do animal.

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
>**Observação:** No exemplo acima, não há definição de rota, ou lógica de negócio, apenas define os tipos de dados pertencentes ao objeto

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

**Componentes UI (ex: `DefaultLoggedPageLayout.tsx`):** O layout define a estrutura fixa (Header, Menu), mas é "aberto" para receber qualquer conteúdo via `children`. Não é necessário alterar o código do layout para criar uma nova página que o utilize.

```typescript
// frontend/src/layout/DefaultLoggedPage/DefaultLoggedPageLayout.tsx
export function DefaultLoggedPageLayout({
  children,
}: {
  children: React.ReactNode
}) {
  // ... lógica interna do menu (fechada para modificação externa)
  return (
    <S.Wrapper>
      {/* ... Header e Sidebar fixos ... */}
      <S.PageContent>{children}</S.PageContent> 
    </S.Wrapper>
  )
}
```
> **Observação:** O componente aceita novos elementos (conteúdos de página) sem alterar seu código fonte.

**Backend Services (Injeção de Dependência):** O serviço `CreateAnimalService` depende de uma interface `AnimalRepository`. Então ao criar o banco de dados, cria-se uma nova implementação do repositório sem modificar o código do serviço.

```typescript
// backend/src/services/animal/create-animal.ts
export class CreateAnimalService {
  constructor(
    // Aberto para extensão: aceita qualquer classe que implemente AnimalRepository
    private readonly animalRepository: AnimalRepository,
    // ...
  ) {}
}
```
> **Observação:** O serviço não precisa ser modificado se a tecnologia de banco de dados mudar, desde que o contrato da interface seja mantido.

## 3. Liskov Substitution Principle (LSP) - Princípio da Substituição de Liskov

Objetos de uma superclasse devem ser substituíveis por objetos de suas subclasses sem quebrar a aplicação.

### ✅ Aplicação Implícita (Backend)
- Nos serviços (ex: `CreateAnimalService`), as dependências são injetadas via construtor (ex: `AnimalRepository`).
- Isso permite que qualquer implementação concreta de `AnimalRepository` (seja Postgres, Mongo ou um Mock em memória para testes) seja usada sem quebrar o serviço, respeitando o contrato da interface.

**Substituição em Testes:** O exemplo mais claro ocorre nos testes unitários. O serviço `CreateAnimalService` espera receber um `AnimalRepository`. Nos testes, passamos um `MockProxy<AnimalRepository>` (uma "subclasse" simulada). O serviço aceita essa substituição e funciona perfeitamente, provando que o princípio está sendo respeitado.

**1. Definição da Classe (O Contrato):**
```typescript
// backend/src/services/animal/create-animal.ts
export class CreateAnimalService {
  constructor(
    // O serviço aceita a classe base ou qualquer substituto compatível
    private readonly animalRepository: AnimalRepository,
    // ...
  ) {}
}
```

**2. A Substituição (O Teste):**
```typescript
// backend/src/services/animal/create-animal.spec.ts
const animalRepository = mock<AnimalRepository>() // Cria um substituto (Mock)
const sut = new CreateAnimalService(animalRepository, ...) // Injeta o substituto
// O serviço executa sua lógica sem saber que está usando um mock, validando o LSP.
```

## 4. Interface Segregation Principle (ISP) - Princípio da Segregação de Interface

Muitas interfaces específicas são melhores do que uma interface única geral.

### ✅ Aplicação Positiva

**Frontend (Schemas Específicos):** No formulário de login, definimos uma interface `UserLoginFormData` que contém *apenas* email e senha. Se usássemos uma interface genérica de `User` (que contém nome, endereço, data de criação, etc), o componente de login dependeria de campos que não utiliza, violando o ISP.

```typescript
// frontend/src/app/login/page.tsx
const userLoginFormSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})
// O componente depende apenas desta interface enxuta
export type UserLoginFormData = z.infer<typeof userLoginFormSchema>
```

**Backend (DTOs):** O `CreateAnimalDTO.Params` define estritamente os dados necessários para a criação de um animal. Ele não obriga o serviço a lidar com uma entidade completa de banco de dados (que teria `id`, `created_at`, `updated_at`), mantendo o contrato limpo e segregado.

```typescript
// backend/src/services/animal/create-animal.ts
export namespace CreateAnimalDTO {
  export type Params = {
    name: string
    type: string
    // ... outros campos específicos para criação
  }
}
```

## 5. Dependency Inversion Principle (DIP) - Princípio da Inversão de Dependência

Módulos de alto nível não devem depender de módulos de baixo nível. 

Em vez disso, eles devem depender de abstrações ou interfaces que definem contratos de funcionamento. Para promover maior flexibilidade e facilitar a manutenção do sistema.

### ✅ Aplicação Positiva (Backend)

**Injeção de Dependência no Construtor:** A classe `CreateAnimalService` não cria suas dependências (banco de dados) internamente. Ela as recebe via construtor, tipadas como interfaces (`AnimalRepository`). Isso inverte a dependência: o serviço não depende do banco, depende de um contrato.

```typescript
// backend/src/services/animal/create-animal.ts
export class CreateAnimalService {
  constructor(
    // Depende da Abstração (Interface), não da implementação concreta (PostgresAnimalRepository)
    private readonly animalRepository: AnimalRepository,
    private readonly animalImageRepository: AnimalImageRepository,
  ) {}
}
```



# Padrões de Projeto - Projeto Adote Fácil

## 1. Singleton

**Onde é usado:** No Backend, na instanciação de Controllers e Services.

**Explicação:** O padrão Singleton garante que uma classe tenha apenas uma instância e fornece um ponto global de acesso a ela.
1. **Mecanismo:** No Node.js, ao exportar uma instância criada com `new`, o sistema de módulos faz cache do objeto.
2. **Uso:** O padrão se confirma pelo fato de que **apenas essa instância específica é importada e utilizada em todo o código**, garantindo que não existam duplicatas de estado ou conexões.

```typescript
// Exemplo inferido da estrutura de rotas
// O controller é instanciado uma única vez e reutilizado em todas as requisições
export const createUserControllerInstance = new CreateUserController(createUserService)
```



## 2. Facade 

O componente consome uma interface simples, ignorando a complexidade do Axios 

**Exemplo (`frontend/src/components/UpdateUserInfoForm/UpdateUserInfoForm.tsx `):**
```typescript
// Toda a complexidade do Axios está sob "fachada" no método updateUser
 const response = await updateUser(data, token)
```

> **Observação:** O uso acima promove organização e clareza no código, facilitando sua posterior implementação e manutenção.