---
description: Règles pour la couche Présentation (controllers, DTOs, interceptors, mappers, Swagger)
globs: src/**/presentation/**/*.ts
---

# Couche Présentation

## HTTP Controllers

**PASSIF et MINCE** : reçoit DTO → exécute command/query → retourne DTO.

### Interdictions :

- AUCUNE logique métier
- AUCUNE transformation complexe
- AUCUN Logger
- AUCUN try/catch

### Fichier : `[module]-http.controller.ts` à la racine de `presentation/`

```typescript
@ApiTags('MyResource')
@Controller('v1/my-resource')
@UseInterceptors(MyHttpDomainExceptionsInterceptor)
export class MyHttpController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() dto: CreateMyRequestDto): Promise<CreateMyResponseDto> {
    return this.commandBus.execute(new CreateMyCommand(dto));
  }

  @Get(':id')
  @HttpCode(HttpStatus.OK)
  findOne(@Param('id') id: string): Promise<MyResponseDto> {
    return this.queryBus.execute(new GetMyQuery({ id }));
  }

  @Put(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  update(@Param('id') id: string, @Body() dto: UpdateMyRequestDto): Promise<void> {
    return this.commandBus.execute(new UpdateMyCommand({ id, ...dto }));
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  delete(@Param('id') id: string): Promise<void> {
    return this.commandBus.execute(new DeleteMyCommand({ id }));
  }
}
```

HTTP codes : 201 (create), 200 (read), 204 (update/delete).

## DTOs

- Objets **PASSIFS** : propriétés + validateurs optionnels + `static fromReadModel()` optionnel
- **INTERDIT** : méthodes d'instance, logique métier, `toCommand()`, `toEntity()`
- Validation `class-validator` sur Request DTOs uniquement
- Swagger : `@ApiProperty`/`@ApiPropertyOptional` avec descriptions
- Pas de terminologie "ViewModel", toujours "DTO"

## Interceptors (Domain Exceptions → HTTP)

```typescript
export class MyHttpDomainExceptionsInterceptor extends HttpDomainExceptionsInterceptor {
  constructor() {
    super();
    this.toHttpException(MyNotFoundException, (e) => new NotFoundException(e.message));
    this.toHttpException(MyAlreadyExistsException, (e) => new ConflictException(e.message));
  }
}
```

## Mappers

- Étendent `BaseMapper<Source, Target[, Options]>`
- `protected map()` + `public from()` + `fromList()`
- `@Injectable()`

## Connector RPC Controllers (Kafka)

- Fichier : `[module]-connector-rpc.controller.ts`
- Étend `KafkaRpcController`
- `@MessagePattern` + `@EventContentTypePattern`
- Validation Zod obligatoire avant commande
- Error handling : `'dead-letter-queue'` ou `'reply'`
- Mapping trivial = passe-plat, non-trivial = mapper dédié
- JAMAIS de logique dans le controller

## Swagger

- `@ApiTags`, `@ApiOperation`, `@ApiParam`, `@ApiQuery`, `@ApiBody`
- Réponses : `@ApiCreatedResponse`, `@ApiOkResponse`, `@ApiNoContentResponse`, `@ApiBadRequestResponse`, `@ApiNotFoundResponse`, `@ApiConflictResponse`
- Format UUID : `format: 'uuid'`
