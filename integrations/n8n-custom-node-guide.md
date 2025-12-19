# Building a Custom FLTR n8n Node

This guide walks through creating a custom n8n community node for FLTR, making it easier for users to integrate FLTR RAG into their workflows.

## Why Build a Custom Node?

**Current State**: Users manually configure HTTP Request nodes ✅ Works but tedious

**With Custom Node**:
- ✅ Visual query builder
- ✅ Dataset autocomplete dropdown
- ✅ Built-in credential management
- ✅ Pre-configured operations
- ✅ Better error messages
- ✅ Type safety and validation

---

## Project Setup

### 1. Initialize Node Package

```bash
# Create project directory
mkdir n8n-nodes-fltr
cd n8n-nodes-fltr

# Initialize npm package
npm init -y

# Install n8n node development dependencies
npm install n8n-workflow n8n-core --save-peer
npm install @types/node typescript --save-dev

# Create TypeScript config
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2019",
    "module": "commonjs",
    "lib": ["ES2019"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
EOF
```

### 2. Update package.json

```json
{
  "name": "n8n-nodes-fltr",
  "version": "1.0.0",
  "description": "n8n node for FLTR document search and RAG",
  "license": "MIT",
  "homepage": "https://fltr.com",
  "author": {
    "name": "FLTR",
    "email": "support@fltr.com"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/n8n-nodes-fltr.git"
  },
  "keywords": [
    "n8n",
    "n8n-community-node-package",
    "fltr",
    "rag",
    "vector-search",
    "semantic-search"
  ],
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "format": "prettier --write \"src/**/*.ts\"",
    "lint": "eslint src --ext .ts"
  },
  "files": [
    "dist"
  ],
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": [
      "dist/credentials/FltrApi.credentials.js"
    ],
    "nodes": [
      "dist/nodes/Fltr/Fltr.node.js"
    ]
  },
  "peerDependencies": {
    "n8n-workflow": "*",
    "n8n-core": "*"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "typescript": "^5.0.0",
    "prettier": "^3.0.0",
    "eslint": "^8.0.0"
  }
}
```

---

## Create Node Files

### 3. Credential Configuration

Create `src/credentials/FltrApi.credentials.ts`:

```typescript
import {
    IAuthenticateGeneric,
    ICredentialType,
    INodeProperties,
} from 'n8n-workflow';

export class FltrApi implements ICredentialType {
    name = 'fltrApi';
    displayName = 'FLTR API';
    documentationUrl = 'https://docs.fltr.com/authentication/api-keys';
    properties: INodeProperties[] = [
        {
            displayName: 'API Key',
            name: 'apiKey',
            type: 'string',
            typeOptions: {
                password: true,
            },
            default: '',
            required: true,
            description: 'Your FLTR API key. Get it from https://app.fltr.com/settings/api-keys',
        },
        {
            displayName: 'API URL',
            name: 'apiUrl',
            type: 'string',
            default: 'https://api.fltr.com/v1',
            description: 'FLTR API base URL',
        },
    ];

    authenticate: IAuthenticateGeneric = {
        type: 'generic',
        properties: {
            headers: {
                Authorization: '=Bearer {{$credentials.apiKey}}',
            },
        },
    };
}
```

### 4. Main Node Implementation

Create `src/nodes/Fltr/Fltr.node.ts`:

```typescript
import {
    IExecuteFunctions,
    INodeExecutionData,
    INodeType,
    INodeTypeDescription,
    NodeOperationError,
} from 'n8n-workflow';

export class Fltr implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'FLTR',
        name: 'fltr',
        icon: 'file:fltr.svg',
        group: ['transform'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Search documents using FLTR RAG',
        defaults: {
            name: 'FLTR',
        },
        inputs: ['main'],
        outputs: ['main'],
        credentials: [
            {
                name: 'fltrApi',
                required: true,
            },
        ],
        properties: [
            // Resource selector
            {
                displayName: 'Resource',
                name: 'resource',
                type: 'options',
                noDataExpression: true,
                options: [
                    {
                        name: 'Dataset',
                        value: 'dataset',
                    },
                    {
                        name: 'Query',
                        value: 'query',
                    },
                ],
                default: 'query',
            },

            // ============= QUERY OPERATIONS =============
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: {
                    show: {
                        resource: ['query'],
                    },
                },
                options: [
                    {
                        name: 'Search',
                        value: 'search',
                        description: 'Search documents in a dataset',
                        action: 'Search documents',
                    },
                    {
                        name: 'Batch Search',
                        value: 'batchSearch',
                        description: 'Search multiple queries at once',
                        action: 'Batch search documents',
                    },
                ],
                default: 'search',
            },

            // Search operation fields
            {
                displayName: 'Dataset ID',
                name: 'datasetId',
                type: 'string',
                required: true,
                displayOptions: {
                    show: {
                        resource: ['query'],
                        operation: ['search'],
                    },
                },
                default: '',
                placeholder: 'ds_abc123',
                description: 'The ID of the dataset to search',
            },
            {
                displayName: 'Query',
                name: 'query',
                type: 'string',
                required: true,
                displayOptions: {
                    show: {
                        resource: ['query'],
                        operation: ['search'],
                    },
                },
                default: '',
                description: 'The search query',
            },
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                displayOptions: {
                    show: {
                        resource: ['query'],
                        operation: ['search'],
                    },
                },
                default: 5,
                typeOptions: {
                    minValue: 1,
                    maxValue: 20,
                },
                description: 'Number of results to return (1-20)',
            },
            {
                displayName: 'Search Mode',
                name: 'searchMode',
                type: 'options',
                displayOptions: {
                    show: {
                        resource: ['query'],
                        operation: ['search'],
                    },
                },
                options: [
                    {
                        name: 'Hybrid (Recommended)',
                        value: 'hybrid',
                        description: 'Combines vector and keyword search for best results',
                    },
                    {
                        name: 'Vector Only',
                        value: 'vector',
                        description: 'Semantic similarity search',
                    },
                    {
                        name: 'Keyword Only',
                        value: 'keyword',
                        description: 'Traditional keyword matching',
                    },
                ],
                default: 'hybrid',
                description: 'Search algorithm to use',
            },
            {
                displayName: 'Enable Reranking',
                name: 'enableReranking',
                type: 'boolean',
                displayOptions: {
                    show: {
                        resource: ['query'],
                        operation: ['search'],
                    },
                },
                default: true,
                description: 'Whether to use Cohere reranking for better relevance',
            },

            // ============= DATASET OPERATIONS =============
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: {
                    show: {
                        resource: ['dataset'],
                    },
                },
                options: [
                    {
                        name: 'List',
                        value: 'list',
                        description: 'List all available datasets',
                        action: 'List datasets',
                    },
                    {
                        name: 'Get Manifest',
                        value: 'getManifest',
                        description: 'Get MCP manifest for a dataset',
                        action: 'Get dataset manifest',
                    },
                ],
                default: 'list',
            },
            {
                displayName: 'Dataset ID',
                name: 'datasetId',
                type: 'string',
                required: true,
                displayOptions: {
                    show: {
                        resource: ['dataset'],
                        operation: ['getManifest'],
                    },
                },
                default: '',
                placeholder: 'ds_abc123',
                description: 'The ID of the dataset',
            },
            {
                displayName: 'Category',
                name: 'category',
                type: 'string',
                displayOptions: {
                    show: {
                        resource: ['dataset'],
                        operation: ['list'],
                    },
                },
                default: '',
                description: 'Filter datasets by category',
            },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        const items = this.getInputData();
        const returnData: INodeExecutionData[] = [];

        const credentials = await this.getCredentials('fltrApi');
        const apiUrl = credentials.apiUrl as string;

        const resource = this.getNodeParameter('resource', 0) as string;
        const operation = this.getNodeParameter('operation', 0) as string;

        for (let i = 0; i < items.length; i++) {
            try {
                if (resource === 'query') {
                    if (operation === 'search') {
                        // Get parameters
                        const datasetId = this.getNodeParameter('datasetId', i) as string;
                        const query = this.getNodeParameter('query', i) as string;
                        const limit = this.getNodeParameter('limit', i) as number;
                        const searchMode = this.getNodeParameter('searchMode', i) as string;
                        const enableReranking = this.getNodeParameter('enableReranking', i) as boolean;

                        // Build query URL
                        const queryParams = new URLSearchParams({
                            datasetId,
                            query,
                            limit: limit.toString(),
                            search_mode: searchMode,
                            enable_reranking: enableReranking.toString(),
                        });

                        // Make request
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${apiUrl}/mcp/query?${queryParams}`,
                            returnFullResponse: false,
                        });

                        // Return results
                        returnData.push({
                            json: response,
                            pairedItem: { item: i },
                        });
                    } else if (operation === 'batchSearch') {
                        // Batch search implementation
                        const queries = this.getNodeParameter('queries', i) as Array<{
                            datasetId: string;
                            query: string;
                            limit?: number;
                        }>;

                        const response = await this.helpers.httpRequest({
                            method: 'POST',
                            url: `${apiUrl}/mcp/batch-query`,
                            body: { queries },
                            returnFullResponse: false,
                        });

                        returnData.push({
                            json: response,
                            pairedItem: { item: i },
                        });
                    }
                } else if (resource === 'dataset') {
                    if (operation === 'list') {
                        const category = this.getNodeParameter('category', i, '') as string;

                        let url = `${apiUrl}/mcp/datasets`;
                        if (category) {
                            url += `?category=${encodeURIComponent(category)}`;
                        }

                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url,
                            returnFullResponse: false,
                        });

                        returnData.push({
                            json: response,
                            pairedItem: { item: i },
                        });
                    } else if (operation === 'getManifest') {
                        const datasetId = this.getNodeParameter('datasetId', i) as string;

                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${apiUrl}/mcp/manifest/${datasetId}`,
                            returnFullResponse: false,
                        });

                        returnData.push({
                            json: response,
                            pairedItem: { item: i },
                        });
                    }
                }
            } catch (error) {
                if (this.continueOnFail()) {
                    returnData.push({
                        json: {
                            error: error.message,
                        },
                        pairedItem: { item: i },
                    });
                    continue;
                }
                throw new NodeOperationError(this.getNode(), error, {
                    itemIndex: i,
                });
            }
        }

        return [returnData];
    }
}
```

### 5. Node Icon

Create `src/nodes/Fltr/fltr.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none">
  <rect width="24" height="24" rx="4" fill="#6366f1"/>
  <path d="M8 6h8M8 10h6M8 14h4" stroke="white" stroke-width="2" stroke-linecap="round"/>
  <circle cx="16" cy="16" r="3" fill="white"/>
</svg>
```

### 6. Entry Point

Create `src/index.ts`:

```typescript
export * from './credentials/FltrApi.credentials';
export * from './nodes/Fltr/Fltr.node';
```

---

## Build and Test Locally

### 7. Build the Node

```bash
# Build TypeScript to JavaScript
npm run build

# Link package locally
npm link
```

### 8. Install in Local n8n

```bash
# Create custom extensions directory if it doesn't exist
mkdir -p ~/.n8n/custom
cd ~/.n8n/custom
npm init -y  # Only if package.json doesn't exist

# Link the FLTR node
npm link n8n-nodes-fltr
```

### 9. Start n8n and Test

```bash
# Start n8n
n8n start

# Open browser at http://localhost:5678
# Search for "FLTR" in the node panel
# You should see your custom node!
```

---

## Testing the Node

### Test Case 1: Basic Search

1. Create new workflow
2. Add "Manual Trigger" node
3. Add "FLTR" node (your custom node!)
4. Configure:
   - **Credential**: Add FLTR API credential
   - **Resource**: Query
   - **Operation**: Search
   - **Dataset ID**: `ds_abc123`
   - **Query**: `test search`
   - **Limit**: 5
5. Execute workflow
6. Verify results in output panel

### Test Case 2: List Datasets

1. Add "FLTR" node
2. Configure:
   - **Resource**: Dataset
   - **Operation**: List
3. Execute
4. See all available datasets

---

## Publishing to npm

### 10. Prepare for Publishing

```bash
# Update version
npm version patch  # or minor/major

# Login to npm
npm login

# Publish
npm publish
```

### 11. Package.json Checklist

Ensure these are set correctly:

```json
{
  "name": "n8n-nodes-fltr",
  "version": "1.0.0",
  "description": "n8n node for FLTR document search and RAG",
  "keywords": [
    "n8n",
    "n8n-community-node-package",
    "fltr"
  ],
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/n8n-nodes-fltr.git"
  }
}
```

---

## Advanced Features

### Dataset Autocomplete (Dynamic Options)

Add dynamic dataset loading to the node:

```typescript
// In Fltr.node.ts, add to properties array:
{
    displayName: 'Dataset',
    name: 'datasetId',
    type: 'options',
    typeOptions: {
        loadOptionsMethod: 'getDatasets',
    },
    required: true,
    default: '',
    description: 'The dataset to search',
}

// Add methods section:
methods = {
    loadOptions: {
        async getDatasets(this: ILoadOptionsFunctions) {
            const credentials = await this.getCredentials('fltrApi');
            const apiUrl = credentials.apiUrl as string;

            const response = await this.helpers.httpRequest({
                method: 'GET',
                url: `${apiUrl}/mcp/datasets`,
            });

            return response.datasets.map((dataset: any) => ({
                name: dataset.name,
                value: dataset.id,
            }));
        },
    },
};
```

### Error Handling

Add better error messages:

```typescript
try {
    // ... API call
} catch (error) {
    if (error.statusCode === 429) {
        throw new NodeOperationError(
            this.getNode(),
            'Rate limit exceeded. Please wait before retrying.',
            { itemIndex: i }
        );
    } else if (error.statusCode === 402) {
        throw new NodeOperationError(
            this.getNode(),
            'Insufficient credits. Please add credits to your account.',
            { itemIndex: i }
        );
    }
    throw error;
}
```

---

## Project Structure

```
n8n-nodes-fltr/
├── package.json
├── tsconfig.json
├── README.md
├── LICENSE
├── src/
│   ├── index.ts
│   ├── credentials/
│   │   └── FltrApi.credentials.ts
│   └── nodes/
│       └── Fltr/
│           ├── Fltr.node.ts
│           └── fltr.svg
└── dist/          (generated by build)
    ├── index.js
    ├── credentials/
    │   └── FltrApi.credentials.js
    └── nodes/
        └── Fltr/
            ├── Fltr.node.js
            └── fltr.svg
```

---

## Troubleshooting

### Node Not Appearing

1. Check `package.json` has correct `n8n` section
2. Verify build ran successfully: `npm run build`
3. Check dist/ directory exists with .js files
4. Restart n8n after linking

### Credential Issues

1. Verify credential file path in package.json
2. Check credential class name matches displayName
3. Clear n8n cache: `n8n clear`

### API Errors

1. Test API directly with curl first
2. Check credential is set correctly
3. Enable debug logging: `N8N_LOG_LEVEL=debug n8n start`

---

## Resources

- [n8n Node Development Docs](https://docs.n8n.io/integrations/creating-nodes/)
- [n8n Community Nodes](https://www.npmjs.com/search?q=keywords:n8n-community-node-package)
- [FLTR API Documentation](https://docs.fltr.com)
- [Example Nodes](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes)

---

## Next Steps

1. ✅ Build basic node (this guide)
2. Test locally with real FLTR API
3. Add advanced features (autocomplete, validation)
4. Write tests
5. Create documentation/README
6. Publish to npm
7. Submit to n8n community nodes registry

---

## Maintenance

### Updating the Node

```bash
# Make changes to src/
npm run build
npm version patch
npm publish
```

### User Installation

Once published, users can install with:

```bash
npm install n8n-nodes-fltr
```

Or via n8n UI:
- Settings → Community Nodes
- Install: `n8n-nodes-fltr`

---

**Estimated Development Time**: 4-6 hours for basic node, 2-3 days for fully-featured version

**Priority**: Medium - HTTP Request nodes work fine, but custom node provides better UX
