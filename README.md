### Добавить NoSQL базу в микросервис

#### Mongo
В качестве NoSQL базы можно использовать Mongo и перенести туда данные и работу с ними для одного из микросервисов на выбор: модуля Администрирование либо модуля Предложение промокодов клиентам.
Mongo образ нужно будет добавить в docker-compose: image: "mongo:4.2.3"

Добавил:
```cs
 #Administration MongoDb
  promocode-factory-administration-mongo-db:
    image: "mongo:4.2.3"
    container_name: 'promocode-factory-administration-mongo-db'
    restart: always 
    ports:
      - 27017:27017
    environment: 
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
```

#### Модуль Администрирование

В модуле Администрирование (Otus.Teaching.Pcf.Administration) нужно изменить хранение сущностей Employee и Role в MongoDb, реализовать новый репозиторий для работы с Mongo, плюс обеспечить работу текущих контроллеров с новым хранилищем.

Добавил репозиторий работы с Mongo
```cs
 public class MongoRepository<T> : IRepository<T> where T : BaseEntity
 {
     private readonly IMongoCollection<T> _collection;
     public MongoRepository(IMongoDatabase mongoDatabase)
     {
         _collection = mongoDatabase.GetCollection<T>(typeof(T).Name);
     }
     public async Task AddAsync(T entity)
     {
         await _collection.InsertOneAsync(entity);
     }

     public async Task DeleteAsync(T entity)
     {
         await _collection.DeleteOneAsync(e => e.Id == entity.Id);
     }

     public async Task<IEnumerable<T>> GetAllAsync()
     {
         return (await _collection.FindAsync(_ => true)).ToEnumerable();
     }

     public async Task<T> GetByIdAsync(Guid id)
     {
         return (await _collection.FindAsync(e => e.Id == id)).FirstOrDefault();
     }

     public async Task<T> GetFirstWhere(Expression<Func<T, bool>> predicate)
     {
         return (await _collection.FindAsync<T>(predicate)).FirstOrDefault();
     }

     public async Task<IEnumerable<T>> GetRangeByIdsAsync(List<Guid> ids)
     {
         return (await _collection.FindAsync(e => ids.Contains(e.Id))).ToEnumerable();
     }

     public async Task<IEnumerable<T>> GetWhere(Expression<Func<T, bool>> predicate)
     {
         return (await _collection.FindAsync(predicate)).ToEnumerable();
     }

     public async Task UpdateAsync(T entity)
     {
         await _collection.ReplaceOneAsync(e => e.Id == entity.Id, entity);
     }
 }

```
В фале Startup.cs добавил Mongo
```cs
services.AddSingleton<IMongoClient>(_ => new MongoClient(Configuration["MongoDb:ConnectionString"]))
    .AddSingleton(serviceProvider => serviceProvider.GetRequiredService<IMongoClient>().GetDatabase(Configuration["MongoDb:DatabaseName"]))
    .AddScoped(serviceProvider => serviceProvider.GetRequiredService<IMongoClient>().StartSession());

services.AddScoped(typeof(IRepository<>), typeof(MongoRepository<>));
services.AddScoped<IDbInitializer, MongoDbInitializer>();
```

Контроллеры, которые с Mongo работают:

<image src="images/res.png" alt="res">

