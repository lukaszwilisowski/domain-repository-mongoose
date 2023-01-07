# domain-repository-example

## How to use domain-repository

### 1. Define your domain models

Make sure you have domain models defined. Each model should be exported in two versions:

- `Detached` (default model without id), for objects not yet persisted in the database
- `Attached` (with id), for already persisted objects

This differentiation improves intellisense and debugging. You can call your models whatever you like, as long as you stick to your naming convention. Our recommendation is to add _Attached suffix_ to all of your attached models.

For example:

```typescript
//We recommend to use IType naming convention for pure model types, to distinguish them from classes
export type ICar = {
  name: string;
  best: boolean;
  readonly yearOfProduction: number;
  sold?: Date;
};

export type ICarAttached = ICar & { id: string };
```

An attached model will contain:

- at minimum, a string Id [(why Id is of type string?)](https://github.com/lukaszwilisowski/domain-repository/blob/main/DISCUSSION.md#6-why-object-id-should-be-of-type-string)
- other properties auto-generated by the db engine (not manually assignable)

---

### 2. Use IDomainRepository in your business services

Use `IDomainRepository`\* interface in places, where you would previously use Mongoose collection or TypeORM repository. Type it explicitly with your standard and Attached model type.

\*If you only need to read or write data you can also use narrowed versions of interfaces: `IReadDomainRepository` or `IWriteDomainRepository` (SOLID's Interface segregation principle).

```typescript
import { IDomainRepository } from 'domain-repository';

export class CarService {
  constructor(private readonly carRepository: IDomainRepository<ICar, ICarAttached>) {}

  public async create(car: ICar): Promise<ICarAttached> {
    return this.carRepository.create(car);
  }

  public async findBestCar(): Promise<ICarAttached | undefined> {
    return this.carRepository.findOne({ best: true });
  }
}
```

---

### 3. Write unit tests (Test-driven-development)

First test your domain model and business service, using MockedDbRepository implementation.

```typescript
import { MockedDBRepository } from 'domain-repository';

describe('CarService', () => {
  const initialData: ICarAttached[] = [
    { id: '1', name: 'Volvo', best: false, yearOfProduction: 2000 },
    {
      id: '2',
      name: 'Toyota',
      best: true,
      yearOfProduction: 2010,
      sold: new Date()
    }
  ];

  const mockedRepository = new MockedDBRepository<ICar, ICarAttached>(initialData);
  const carService = new CarService(mockedRepository);

  it('should find best car', async () => {
    const car = await carService.findBestCar();

    expect(car).toBeDefined();
    expect(car!.name).toEqual('Toyota');
  });
});
```

---

### 4. Choose your DB technology and define model mappings.

Let's say I want to use MongoDB as my DB, and Mongoose as my ORM layer.

Let's create a new file for my DB model, for example: `car.entity.ts`:
Because we have mappings, this does not have to be the same model as domain model.

```typescript
export type ICarMongoEntity = {
  _id: mongoose.Types.ObjectId;
  name: string;
  best_of_all: boolean;
  readonly yearOfProduction: number;
  sold?: Date;
};
```

Now create file `car.schema.ts` and define your db schema, using mongoose:

```typescript
import { Mapping } from 'domain-repository/mapping';
import { mapToMongoObjectId } from 'domain-repository/db/mongodb';

//standard MongoDb schema, typed with your db model
export const CarSchema = new Schema<ICarMongoEntity>({
  name: {
    type: String,
    required: true
  },
  best_of_all: {
    type: Boolean,
    required: true
  },
  yearOfProduction: {
    type: Number,
    required: true
  },
  sold: {
    type: Date,
    required: false
  }
});

//define mapping from domain model to db model
export const mongoCarMapping: Mapping<ICarAttached, ICarMongoEntity> = {
  id: mapToMongoObjectId,
  name: 'name',
  best: 'best_of_all',
  yearOfProduction: 'yearOfProduction',
  sold: 'sold'
};
```

If you are interested, `mapToMongoObjectId` has a simple implementation:

```typescript
export const mapToMongoObjectId: TransformProperty<'_id', string, mongoose.Types.ObjectId> = MapTo.Property(
  '_id',
  (objectId: string) => new mongoose.Types.ObjectId(objectId),
  (entityId: mongoose.Types.ObjectId) => entityId.toString()
);
```

Please note that our Mapping allows for more advanced transformations, such as:

- a property can be mapped to other property with compatible type but different name, using direct assignment: `property: 'mappedProperty'`
- a primitive property can be mapped to other primitive property of different type, using transformation helper `MapTo.Property(mappedProperty, transformFunc, reverseTransformFunc)`
- an array of primitives property can be mapped to other array of primitives, using single element transformation helper `MapTo.Array(mappedProperty, transformElementFunc, reverseTransformElementFunc)`
- an object array property can be mapped to other object array property, using nested mapping `MapTo.ObjectArray(mappedProperty, nestedMapping)`
- a nested object property can be mapped to other nested object property, using nested mapping `MapTo.NestedObject(mappedProperty, nestedMapping)`

You can find an example of advanced nested object mapping [here](https://github.com/lukaszwilisowski/domain-repository/blob/main/test/db/mongoose/entities/car/car.entity.ts).

---

### 5. Supply your services with proper repository implemenation for your target DB.

Now depending on your db and ORM layer, you need to create ORM repository and pass it to our implementation of IDomainRepository.

MongoDb example:

```typescript
import { MongoDbRepository } from 'domain-repository/db/mongodb';

const runMongoTest = async (): Promise<void> => {
  await new Promise<void>((resolve) => {
    mongoose.connect('mongodb://localhost:27017/testdb', {});
    mongoose.connection.on('open', () => resolve());
  });

  const carRepository = new MongoDbRepository<ICar, ICarAttached, ICarMongoEntity>(
    mongoose.model<ICarMongoEntity>('cars', CarSchema),
    mongoCarMapping
  );

  const carService = new CarService(carRepository);

  await carService.create({
    name: 'Toyota',
    best: true,
    yearOfProduction: 2010,
    sold: new Date()
  });

  const bestCar = await carService.findBestCar();
  console.log(bestCar);
};

runMongoTest();
```

Output:

```bash
{
  id: '63b8091cdd1f0c4927ca4725',
  name: 'Toyota',
  best: true,
  yearOfProduction: 2010,
  sold: 2023-01-06T11:42:20.836Z
}
```

MongoDB data (see best_of_all renamed property):

```json
{
  "_id": {
    "$oid": "63b8091cdd1f0c4927ca4725"
  },
  "name": "Toyota",
  "best_of_all": true,
  "yearOfProduction": 2010,
  "sold": {
    "$date": "2023-01-06T11:42:20.836Z"
  },
  "__v": 0
}
```

---

PostgreSQL example:

```typescript
import { PostgreSQLDbRepository } from 'domain-repository/db/postgresql';

const runPostgresTest = async (): Promise<void> => {
  const dataSource = new DataSource({
    type: 'postgres',
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    username: 'postgres',
    password: 'admin',
    synchronize: true, //for local testing
    entities: [SqlCarEntity]
  });

  await dataSource.initialize();

  const carRepository = new PostgreSQLDbRepository<ICar, ICarAttached, ICarSqlEntity>(
    dataSource.getRepository(SqlCarEntity),
    sqlCarMapping
  );

  const carService = new CarService(carRepository);

  await carService.create({
    name: 'Toyota',
    best: true,
    yearOfProduction: 2010,
    sold: new Date()
  });

  const bestCar = await carService.findBestCar();
  console.log(bestCar);
};

runPostgresTest();
```

Output:

```bash
{
  id: '146',
  name: 'Toyota',
  best: true,
  yearOfProduction: 2010,
  sold: '2023-01-06T13:11:43.685+01:00'
}
```

PostgreSQL data (see best_of_all renamed property):

```
id,"name","best_of_all","yearOfProduction","sold"
146,"Toyota",True,2010,"2023-01-06T13:11:43.685+01:00"
```
