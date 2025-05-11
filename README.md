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