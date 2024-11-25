# feat-add-url-param-custom_id

## What this feat does: 
> Allow you to add a url param as your custom_id to identify users, as an alternative compared with original UUID.

<img width="1109" alt="image" src="https://github.com/user-attachments/assets/b2817624-4ee6-4f08-9403-b780a7fb4190">

Tests passes in: 

- Simeple Chat: custom_id displayed in Dify log page and Langfuse
- Chatflow: custom_id displayed in Dify log page and Langfuse

## Code changes:

### Frontend - Add url params for /password
> dify\web\service\share.ts
```typescript
export const fetchAccessToken = async (appCode: string) => {
  const headers = new Headers()
  headers.append('X-App-Code', appCode)
  

  const urlParams = new URLSearchParams(window.location.search)
  const custom_id = urlParams.get('custom_id')
  // const username = urlParams.get('custom_id')

  const params: Record<string, string> = {}
  if (custom_id)
    params.custom_id = custom_id

  return get('/passport', { 
    headers,
    params 
  }) as Promise<{ access_token: string }>
}
```

### Frontend - Display end_user name
> dify\web\app\components\app\log\list.tsx
```typescript
const endUser = log.from_account_name || log.from_end_user_name || log.from_end_user_session_id
```

### Backend - passport api
Load `custom_id` param and save it in existed `name` col of `EndUser` table
> dify\api\controllers\web\passport.py
```python
        if request.args.get('custom_id'):
            end_user = EndUser(
                tenant_id=app_model.tenant_id,
                app_id=app_model.id,
                type="browser",
                is_anonymous=False,
                session_id=generate_session_id(),
                name=request.args.get('custom_id')
            )
        else:
            end_user = EndUser(
                tenant_id=app_model.tenant_id,
                app_id=app_model.id,
                type="browser",
                is_anonymous=True,
                session_id=generate_session_id(),
            )

```
### Backend - conversation api
Attach `name` col based on existed `conversation` model output (not the best solution)
> dify\api\controllers\console\app\conversation.py
```python
        conversations = db.paginate(query, page=args["page"], per_page=args["limit"], error_out=False)

        for item in conversations.items:
            if isinstance(item, tuple):
                conversation, from_end_user_name = item
                conversation.from_end_user_name = from_end_user_name
            else:
                end_user = db.session.query(EndUser).filter(EndUser.id == item.from_end_user_id).first()
                item.from_end_user_name = end_user.name if end_user else None

        
        return conversations
```
### Celery - update trace metadata
Notes: replace `user_id` of langfuse with `custom_id` (like user123, 123) will cause error. The `user_id` needs complexity. Currently the `custom_id` will be passed into metadata.

1. Fetch `custom_id` value in table using original value (session_id)
> dify\api\core\ops\langfuse_trace\langfuse_trace.py
```python
        end_user_data: EndUser = (
                db.session.query(EndUser).filter(EndUser.id == user_id).first()
            )
```
2. Update metadata
> dify\api\core\ops\langfuse_trace\langfuse_trace.py
```python
                metadata={
                    **trace_info.metadata,
                    'custom_id': end_user_data.name
                },
```
