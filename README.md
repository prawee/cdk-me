# CDK-ME

## Make folder and repo
```bash

mkdir cdk-me && cd cdk-me
npm init -y
```

## Install dev package
```bash
npm i -D aws-cdk aws-cdk-lib constructs typescript ts-node @types/node
```

## Make files for logic
```bash
mkdir src
mkdir src/infra
touch src/infra/Launcher.ts
mkdir src/infra/stacks
touch src/infra/stacks/DataStack.ts
```

### DataStack
```bash
nano src/infra/stacks/DataStack.ts
```
```
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';

export class DataStack extends Stack {
    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);
    }
}
```

### Launcher
```bash
nano src/infra/Launcher.ts
```
```bash
import { App } from 'aws-cdk-lib';
import { DataStack } from './stacks/DataStack';

const app = new App();
new DataStack(app, 'DataStack');
```

## Make `cdk` configure with `cdk.json`
```bash
touch cdk.json
nano cdk.json
```
```bash
{
    "app": "npx ts-node src/infra/Launcher.ts"
}
```

## Testing `Synth`
```bash
cdk synth
```
### If exist error `TypeError: Unknown file extension ".ts"`
```bash
touch tsconfig.json
nano tsconfig.json
```
```bash
{
    "compilerOptions": {
        "module": "commonjs",
        "target": "es2022"
    }
}
```

## Make Service
### Create Lambda func
```bash
mkdir src/services
touch src/services/hello.js
nano src/services/hello.js
```
```bash
exports.main = async function(event, context) {
    return {
        statusCode: 200,
        body: JSON.stringify('Hello World from Lambda!')
    }
}
```

### Create more Stack for Lambda
```bash
touch src/infra/stacks/LambdaStack.ts
nano src/infra/stacks/LambdaStack.ts
```
```bash
import { Stack, StackProps } from 'aws-cdk-lib';
import { Function as LambdaFunction, Runtime, Code } from 'aws-cdk-lib/aws-lambda';
import { Construct } from 'constructs';
import { join } from 'path';

export class LambdaStack extends Stack {
    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);

        new LambdaFunction(this, 'HelloLambda', {
            runtime: Runtime.NODEJS_20_X,
            handler: 'hello.main',
            code: Code.fromAsset(join(__dirname, '../../services')),
        });
    }
}
```

### Register service to app
```bash
nano src/infra/Launcher.ts
```
```bash
...
import { LambdaStack } from './stacks/LambdaStack';

const app = new App();
...
new LambdaStack(app, 'LambdaStack');
```

## Deploy
```bash
cdk synth
cdk deploy --all
```

### Destroy
```bash
cdk destroy --all --force
```

## Make REST API
### Clone from `DataStack` and update class name
```bash
cp src/infra/stacks/DataStack.ts src/infra/stacks/ApiStack.ts
nano src/infra/stacks/ApiStack.ts
```
```bash
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';

export class ApiStack extends Stack {
    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);
    }
}
```
### Adding logic with `RestApi`
- create `api` from RestApi
- create root api
- adding resource and integration lambda
- update `props` via extends and new define with data integration

```bash
import { Stack, StackProps } from 'aws-cdk-lib';
import { LambdaIntegration, RestApi } from 'aws-cdk-lib/aws-apigateway';
import { Construct } from 'constructs';

interface ApiStackProps extends StackProps {
    helloLambdaIntegration: LambdaIntegration
}
export class ApiStack extends Stack {
    constructor(scope: Construct, id: string, props: ApiStackProps) {
        super(scope, id, props);

        const api = new RestApi(this, 'DemoApi');
        const demoResource = api.root.addResource('demo');
        demoResource.addMethod('GET', props.helloLambdaIntegration);
    }
}
```

### Update `LambdaStack` for support integration
- update `func` with variable
- create public `variable for integration` with `LambdaIntegration` type
- using `variable of integration` with `func`

```bash
...
import { LambdaIntegration } from 'aws-cdk-lib/aws-apigateway';
...

export class LambdaStack extends Stack {

    public readonly helloLambdaIntegration: LambdaIntegration;

    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);

        const helloLambda = new LambdaFunction(this, 'HelloLambda', {
            runtime: Runtime.NODEJS_20_X,
            handler: 'hello.main',
            code: Code.fromAsset(join(__dirname, '../../services')),
        });

        this.helloLambdaIntegration = new LambdaIntegration(helloLambda);
    }
}
```

### Update `Launcher` with integration
- create variable for `LambdaStack`
- register `ApiStack` to app and set props with integration value

```bash
nano src/infra/Launcher.ts
```
```bash
...
import { ApiStack } from './stacks/ApiStack';

const app = new App();
...
const lambdaStack = new LambdaStack(app, 'LambdaStack');
new ApiStack(app, 'ApiStack', {
    helloLambdaIntegration: lambdaStack.helloLambdaIntegration
});
```

## Make Testing
```bash
nano demo.http
```
```bash
# Demo API

#### Getting
GET https://yagw0nmvu2.execute-api.ap-southeast-1.amazonaws.com/prod/demo
```

## Using Table with `DynamoDB`
- create `Utils` for suffix id
- update `DataStack` using `DynamoDB` and export it
- using data with lambda

### Create `Utils` with `getSuffixFromStack` func
```bash
touch src/infra/Utils.ts 
```
```bash
import { Fn, Stack } from 'aws-cdk-lib';

export function getSuffixFromStack(stack: Stack) {
    const shortStackId = Fn.select(2, Fn.split('/', stack.stackId));
    const suffix = Fn.select(4, Fn.split('-', shortStackId));
    return suffix;
}
```

### Update `DataStack` with `DynamoDB`
```bash
nano src/infra/stacks/DataStack.ts
```
```bash
...
import { AttributeType, ITable, Table } from 'aws-cdk-lib/aws-dynamodb';
import { getSuffixFromStack } from '../Utils';

export ... {

    public readonly demoTable: ITable;

    constructor(...) {
        super(...);

        const suffix = getSuffixFromStack(this);

        this.demoTable = new Table(this, 'DemoTable', {
            tableName: `DemoTable-${suffix}`,
            partitionKey: {
                name: 'id',
                type: AttributeType.STRING
            }
        });
    }
}
```