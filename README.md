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