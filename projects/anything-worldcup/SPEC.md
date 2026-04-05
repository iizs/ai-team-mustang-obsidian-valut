# Anything Worldcup — SPEC

## Requirements

**Service Overview**
"Anything Worldcup" is a web service where users compare N items two at a time in tournament format to determine their #1 preference. Admins register topics and items; regular users select a topic and play through the tournament.

**User Types**
- Regular user: select topic → run tournament → view winner. No editing.
- Admin: CRUD for topics and items. No tournament.

**User Page (mobile-first)**
- Display registered topic list with icon images
- Select a topic → tournament starts with that topic's items
- If item count is not 2^n, randomly select the nearest lower 2^n (e.g., 10 items → 8)
- Compare 2 items at a time → click to choose → repeat until winner
- Show final winner screen; allow restart
- No data editing

**Admin Page**
- Login: id/pw (admin/admin)
- Topic CRUD: name + icon image upload
- Item CRUD: name + image upload + random fallback emoji
- No tournament

**Image Handling**
- Uploaded images are resized server-side and stored as files
  - Item images: 400×400px (mobile card size)
  - Topic icons: 100×100px
- If no image provided, a random emoji is used as fallback
- Images served to user pages via backend API

**Technical Requirements**
- Frontend: React (port 8800)
- Backend: Python + venv (port 8801)
- DB: SQLite, abstracted for future migration
- Image storage: file system
- Local execution only

**Non-functional Requirements**
- User page: mobile-first UI/UX
- Tournament results are not persisted (session-only)
- Data editing restricted to admin

---

## Technical Design

**Directory Structure**
```
anything-worldcup/
├── SPEC.md
├── frontend/          # React (Vite)
│   ├── src/
│   │   ├── pages/user/     # Topic list, Tournament, Winner
│   │   ├── pages/admin/    # Login, Topics, Items
│   │   ├── components/
│   │   └── api/            # API client
│   └── package.json
└── backend/           # Python FastAPI
    ├── app/
    │   ├── main.py
    │   ├── models.py       # SQLAlchemy models
    │   ├── database.py     # DB engine (SQLite → swappable)
    │   ├── routers/
    │   │   ├── topics.py   # Public topic/item endpoints
    │   │   └── admin.py    # Admin CRUD + auth endpoints
    │   └── utils/
    │       └── images.py   # Pillow resize logic
    ├── uploads/            # Stored images (gitignored)
    ├── requirements.txt
    └── venv/               # gitignored
```

**API Contract**

Public:
- `GET /api/topics` — topic list (id, name, icon_url, emoji)
- `GET /api/topics/{id}/items` — item list for topic

Admin (requires auth token):
- `POST /api/admin/login` → `{token}`
- `POST/PUT/DELETE /api/admin/topics/{id?}`
- `POST/PUT/DELETE /api/admin/topics/{id}/items / /api/admin/items/{id}`
- `POST /api/admin/topics/{id}/items/{iid}/image`

Static:
- `GET /uploads/{filename}` — serve image files

**DB Schema (SQLAlchemy)**
```
topics: id, name, icon_path, icon_emoji, created_at
items:  id, topic_id, name, image_path, emoji, created_at
```

**Auth**
- JWT token issued on `POST /api/admin/login`
- Hardcoded credentials: admin/admin
- Admin routes protected by token validation middleware

**Frontend Routes**
- `/` — topic list (user)
- `/tournament/:topicId` — tournament (user)
- `/admin` — login
- `/admin/topics` — topic management
- `/admin/topics/:id/items` — item management

**Image Processing**
- Crop to square (center crop), then resize using Pillow
- Item: 400×400px JPEG/WebP
- Icon: 100×100px JPEG/WebP

---

## Success Criteria

### SC-1: User page — topic list
- Given the backend is running and topics exist in the DB,
  When a user visits `/`,
  Then the topic list is displayed with each topic's icon image (or fallback emoji).

### SC-2: Tournament start with 2^n enforcement
- Given a topic with exactly 2^n items (e.g., 8),
  When user selects the topic,
  Then the tournament starts with all 8 items.

- Given a topic with non-2^n items (e.g., 10),
  When user selects the topic,
  Then 8 items are randomly selected and the tournament starts with those 8.

### SC-3: Tournament flow
- Given a tournament is in progress,
  When the match screen is shown,
  Then exactly 2 items are displayed side by side with their image (or emoji fallback).

- Given a match is displayed,
  When the user clicks one item,
  Then the selected item advances and the next match is shown.

- Given only 2 items remain (final),
  When the user picks one,
  Then the winner screen is displayed showing the winning item.

- Given the winner screen is shown,
  When the user chooses to restart,
  Then a new tournament begins with the same topic (re-randomized).

### SC-4: User page is read-only
- Given the user page,
  When any user interaction is tested,
  Then no create, update, or delete of topics or items is possible.

### SC-5: Admin authentication
- Given the admin login page at `/admin`,
  When admin/admin credentials are submitted,
  Then a JWT token is issued and the user is redirected to `/admin/topics`.

- Given an unauthenticated request to any `/api/admin/*` endpoint,
  When the request is made without a valid token,
  Then the response is 401 Unauthorized.

### SC-6: Admin topic CRUD
- Given a logged-in admin,
  When a new topic is created with name + icon image,
  Then the topic appears in the admin topic list and on the user page (`GET /api/topics`).

- Given an existing topic,
  When the admin updates its name or icon,
  Then the changes are reflected in both admin and user views.

- Given an existing topic with items,
  When the admin deletes the topic,
  Then the topic and all its items are removed; it no longer appears on the user page.

### SC-7: Admin item CRUD
- Given a topic in admin,
  When an item is added with a name and image,
  Then the item appears in that topic's item list.

- Given an item with no image uploaded,
  When viewed in the user page tournament,
  Then a random emoji is displayed as the fallback.

- Given an existing item,
  When the admin deletes it,
  Then it no longer appears in the topic's item list or tournament.

### SC-8: Image processing
- Given an admin uploads any image for an item,
  When the upload is processed,
  Then the stored file is exactly 400×400px (center-cropped and resized).

- Given an admin uploads any image for a topic icon,
  When the upload is processed,
  Then the stored file is exactly 100×100px (center-cropped and resized).

- Given an item with a stored image,
  When the user page requests it,
  Then the image is served correctly via `GET /uploads/{filename}`.

### SC-9: Mobile-first layout
- Given the user page loaded on a 390px-wide viewport (iPhone-sized),
  When topic list and tournament screens are rendered,
  Then layout is usable without horizontal scrolling or overflow.

### SC-10: Tournament results not persisted
- Given a user completes a tournament,
  When the session ends or the user navigates away,
  Then no tournament result is stored in the DB.

### SC-11: Text readability on dark background
- Given the user page (topic list, tournament match cards, winner screen),
  When any text is rendered on a dark background,
  Then the text color has sufficient contrast to be clearly readable (WCAG AA minimum: 4.5:1 contrast ratio for normal text).

### SC-12: Topics with insufficient items hidden from user page
- Given a topic with 0 or 1 items exists in the DB,
  When the user page renders the topic list,
  Then that topic does not appear in the list.

- Given a topic with 2 or more items,
  When the user page renders the topic list,
  Then that topic is visible and selectable.

---

## Open Issues

*(None)*

---

## Changelog

- v0.1: Initial spec — React+Python+SQLite architecture, user/admin split
- v0.2: UI text visibility fix, hide topics with < 2 items from user page
