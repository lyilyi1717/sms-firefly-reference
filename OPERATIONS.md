# Operations Reference

## SSH Rule

**NEVER use PowerShell for SSH to Unraid** — triggers Windows Defender trojan false positive.  
Always use the **Bash tool** in Claude Code, or a real terminal (Git Bash / WSL).

```bash
# Correct
ssh root@192.168.8.10 "command here"

# Wrong — DO NOT DO THIS
PowerShell: ssh root@192.168.8.10   ← Windows Defender false positive
```

---

## Dashboard Rebuild

The `sms-dashboard` container has **no volume mount** — all code is baked into the image.  
`docker restart` will NOT pick up `app.js` changes. You must rebuild.

```bash
ssh root@192.168.8.10 "
  cd /mnt/user/appdata/sms-dashboard
  docker build -t sms-dashboard:latest .
  docker stop sms-dashboard
  docker rm sms-dashboard
  docker run -d --name sms-dashboard --restart unless-stopped -p 8283:3000 \
    -e FIREFLY_URL=http://192.168.8.10:8181 \
    -e FIREFLY_TOKEN='<FIREFLY_TOKEN>' \
    -e TELEGRAM_TOKEN='<TELEGRAM_TOKEN>' \
    -e TELEGRAM_CHAT_ID=<TELEGRAM_CHAT_ID> \
    -e PGHOST=192.168.8.10 -e PGPORT=5434 \
    -e PGUSER=<PG_USER> -e PGPASSWORD=<PG_PASS> -e PGDATABASE=n8n \
    sms-dashboard:latest
"
```

Actual values in `SECRETS.md`.

---

## Workflow Export (after any patch)

Always re-export after pushing changes to n8n to keep local JSON files current.

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

Saves:
- `C:\Users\BigB\sms-firefly-wf-IUliVtDElkxMnl9t.json`
- `C:\Users\BigB\sms-firefly-wf-smscorrection01.json`

---

## Patching Workflow

1. Read current workflow JSON: `C:\Users\BigB\sms-firefly-wf-IUliVtDElkxMnl9t.json`
2. Identify target nodes by exact name
3. Write patch script (`patch_<description>.js`) in `C:\Users\BigB\`
4. Run patch: `node C:/Users/BigB/patch_<description>.js`
5. Re-export workflow JSON
6. Verify: check a live execution with `?includeData=true`

---

## Checking Live Executions

```javascript
var http = require('http');
var KEY = '<N8N_API_KEY>';
function apiReq(method, path) {
  return new Promise(function(resolve, reject) {
    var opts = { hostname:'192.168.8.10', port:5678, path:path, method:method,
      headers:{'X-N8N-API-KEY':KEY,'Content-Type':'application/json'} };
    var req = http.request(opts, function(res) {
      var buf=''; res.on('data',function(d){buf+=d;});
      res.on('end',function(){ resolve(JSON.parse(buf)); });
    });
    req.on('error',reject); req.end();
  });
}

// Last 5 executions (summary)
var r = await apiReq('GET', '/api/v1/executions?workflowId=IUliVtDElkxMnl9t&limit=5');
r.data.forEach(function(e) {
  var dur = e.stoppedAt ? (new Date(e.stoppedAt)-new Date(e.startedAt)) : 0;
  console.log(e.id, e.status, Math.round(dur/1000)+'s');
});

// Single execution with full data
var exec = await apiReq('GET', '/api/v1/executions/EXEC_ID?includeData=true');
var nodes = exec.data.resultData.runData;
console.log('Nodes ran:', Object.keys(nodes).join(', '));
```

Execution duration guide:
- `< 3s` = empty queue (no item)
- `3–20s` = regex or merchant hit
- `> 20s` = Ollama AI tier (~42s for gemma3:1b)

---

## PostgreSQL Direct Access

```bash
# Via Docker exec (from Unraid)
ssh root@192.168.8.10 "docker exec -it n8n-postgresql18 psql -U admin -d n8n"

# Queue summary
SELECT status, COUNT(*) FROM sms_queue GROUP BY status;

# Reset stalled items
UPDATE sms_queue SET status='pending', retry_count=0
WHERE status='processing' AND last_attempted_at < NOW()-INTERVAL '15 minutes';
```

---

## Enabling / Disabling Workflow

```javascript
// Disable (stop scheduler)
await apiReq('POST', '/api/v1/workflows/IUliVtDElkxMnl9t/deactivate');

// Enable
await apiReq('POST', '/api/v1/workflows/IUliVtDElkxMnl9t/activate');
```
