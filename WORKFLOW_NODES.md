# n8n Workflow Node Patterns

## Before Any Patch

1. Read `sms-firefly-wf-IUliVtDElkxMnl9t.json` for exact node names, IDs, and current connections
2. Never guess node names from memory — they must match exactly
3. After pushing: re-export JSON (see OPERATIONS.md)
4. Check live execution: `GET /api/v1/executions?workflowId=IUliVtDElkxMnl9t&limit=5&includeData=true`

---

## n8n API Call Template (Node.js)

```javascript
var http = require('http');
var KEY = '<N8N_API_KEY>';  // from SECRETS.md

function apiReq(method, path, body) {
  return new Promise(function(resolve, reject) {
    var data = body ? JSON.stringify(body) : null;
    var opts = {
      hostname: '192.168.8.10', port: 5678,
      path: path, method: method,
      headers: { 'X-N8N-API-KEY': KEY, 'Content-Type': 'application/json' }
    };
    if (data) opts.headers['Content-Length'] = Buffer.byteLength(data);
    var req = http.request(opts, function(res) {
      var buf = '';
      res.on('data', function(d) { buf += d; });
      res.on('end', function() {
        if (res.statusCode >= 400) return reject(new Error('HTTP ' + res.statusCode + ': ' + buf.slice(0, 300)));
        try { resolve(JSON.parse(buf)); } catch(e) { resolve(buf); }
      });
    });
    req.on('error', reject);
    if (data) req.write(data);
    req.end();
  });
}
```

### Get workflow
```javascript
var wf = await apiReq('GET', '/api/v1/workflows/IUliVtDElkxMnl9t');
```

### Push workflow
```javascript
await apiReq('PUT', '/api/v1/workflows/IUliVtDElkxMnl9t', {
  name: wf.name,
  nodes: wf.nodes,          // modified nodes array
  connections: wf.connections,
  settings: wf.settings || {},
  staticData: wf.staticData || null
});
```

---

## Node Type Reference

### Postgres Node (v2.5)

```javascript
{
  type: 'n8n-nodes-base.postgres',
  typeVersion: 2.5,
  credentials: { postgres: { id: 'qx7cEqJpZFgDgCax', name: 'Postgres account' } },
  continueOnFail: true,   // ALWAYS true for volatile nodes — never swallow errors silently
  parameters: {
    operation: 'executeQuery',
    query: 'SELECT ...',
    options: {}
  }
}
```

> **Warning:** `continueOnFail: true` swallows errors silently. After deploy, always inspect execution data.

---

### IF Node (v2) — Condition Structure

```javascript
{
  parameters: {
    conditions: {
      options: {
        caseSensitive: true,
        leftValue: '',
        typeValidation: 'strict',
        version: 1
      },
      conditions: [
        {
          id: 'cond-unique-id',
          leftValue: '={{ $json.field }}',
          rightValue: 'value',
          operator: { type: 'string', operation: 'equals' }
        }
      ],
      combinator: 'and'
    },
    options: {}
  }
}
```

**Operator types:** `string`, `number`, `boolean`  
**Operations:** `equals`, `notEmpty`, `contains`, `greaterThan`, `notEquals`

---

### HTTP Request Node

```javascript
{
  type: 'n8n-nodes-base.httpRequest',
  typeVersion: 4.2,
  parameters: {
    method: 'POST',
    url: 'http://192.168.8.10:8181/api/v1/transactions',
    authentication: 'genericCredentialType',
    genericAuthType: 'httpHeaderAuth',
    sendHeaders: true,
    headerParameters: {
      parameters: [{ name: 'Authorization', value: 'Bearer <FIREFLY_TOKEN>' }]
    },
    sendBody: true,
    bodyParameters: {
      parameters: [
        { name: 'transactions[0][type]', value: '={{ $json.transaction_type }}' },
        { name: 'transactions[0][amount]', value: '={{ $json.amount }}' },
        // ...
      ]
    }
  }
}
```

---

### Code Node — Key Patterns

**Access upstream node output:**
```javascript
// Current item
$json.field

// Specific upstream node (use when $json was overwritten)
$('Parse SMS').item.json.body
$('Config').item.json.banks
```

**n8n expression in SQL query strings:**
```
{{ $json.merchant_safe }}   ← simple field reference (safe)
$('Node').item.json.field   ← NOT safe inside SQL strings in Postgres node query param
```

> **Fix pattern:** When a Code node needs to pass values to a Postgres node query, pre-extract into `$json.v_fieldname` in a Code node immediately before, then use `{{ $json.v_fieldname }}` in the query.

---

## Connections Format

```javascript
connections: {
  'Node Name': {
    main: [
      [{ node: 'Next Node', type: 'main', index: 0 }]   // output 0 → input 0
    ]
  },
  'IF Node': {
    main: [
      [{ node: 'True Branch', type: 'main', index: 0 }],  // output 0 = true
      [{ node: 'False Branch', type: 'main', index: 0 }]  // output 1 = false
    ]
  }
}
```

---

## Checking Execution Results

```javascript
// Last 5 executions
await apiReq('GET', '/api/v1/executions?workflowId=IUliVtDElkxMnl9t&limit=5')

// Single execution with full node data
await apiReq('GET', '/api/v1/executions/EXEC_ID?includeData=true')

// Extract specific node output
var runData = exec.data.resultData.runData;
var nodeOutput = runData['Node Name'][0].data.main[0];
```

---

## Workflow JSON Export

```bash
node -e "
var http=require('http'), fs=require('fs');
var KEY='<N8N_API_KEY>';
['IUliVtDElkxMnl9t','smscorrection01'].forEach(function(id){
  var opts={hostname:'192.168.8.10',port:5678,path:'/api/v1/workflows/'+id,
    method:'GET',headers:{'X-N8N-API-KEY':KEY}};
  var req=http.request(opts,function(res){
    var buf=''; res.on('data',function(d){buf+=d;});
    res.on('end',function(){
      fs.writeFileSync('C:/Users/BigB/sms-firefly-wf-'+id+'.json',
        JSON.stringify(JSON.parse(buf),null,2));
      console.log('saved',id);
    });
  }); req.end();
});
"
```
