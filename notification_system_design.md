from fastapi import FastAPI, Request
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime
import uuid
import time

app = FastAPI()

notifications_db = []
user_notifications = {}

#model
class NotificationCreate(BaseModel):
    title: str
    message: str
    type: str
    target_users: Optional[List[str]] = []

#Middleware (v1)
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    start_time = time.time()

    response = await call_next(request)

    process_time = round((time.time() - start_time) * 1000, 2)

    print({
        "method": request.method,
        "path": request.url.path,
        "status_code": response.status_code,
        "latency_ms": process_time
    })

    return response



#creating notification

@app.post("/notifications")
def create_notification(payload: NotificationCreate):
    notification_id = str(uuid.uuid4())

    notification = {
        "id": notification_id,
        "title": payload.title,
        "message": payload.message,
        "type": payload.type,
        "created_at": datetime.utcnow()
    }

    notifications_db.append(notification)

    
    for user_id in payload.target_users:
        if user_id not in user_notifications:
            user_notifications[user_id] = []

        user_notifications[user_id].append({
            "notification_id": notification_id,
            "read": False
        })

    return {
        "message": "Notification created",
        "id": notification_id
    }



#User notifications

@app.get("/users/{user_id}/notifications")
def get_user_notifications(user_id: str):
    user_data = user_notifications.get(user_id, [])

    result = []

    for entry in user_data:
        notif = next((n for n in notifications_db if n["id"] == entry["notification_id"]), None)
        if notif:
            result.append({
                **notif,
                "read": entry["read"]
            })

    return result



# mark Notification as Read
@app.patch("/users/{user_id}/notifications/{notification_id}/read")
def mark_as_read(user_id: str, notification_id: str):
    user_data = user_notifications.get(user_id, [])

    for entry in user_data:
        if entry["notification_id"] == notification_id:
            entry["read"] = True
            return {"message": "Marked as read"}

    return {"message": "Notification not found"}
