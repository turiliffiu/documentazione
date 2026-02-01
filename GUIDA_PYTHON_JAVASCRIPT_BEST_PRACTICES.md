# GUIDA DI RIFERIMENTO PROGETTI PYTHON E JAVASCRIPT
## Best Practices, Pattern e Comandi Shell dai Progetti Completati

**Data:** 1 Febbraio 2026  
**Utente:** Salvo (turiliffiu)  
**Progetti analizzati:** 20 progetti Python/JavaScript completi

---

# INDICE

1. [Progetti Completi Python/JavaScript](#progetti-completi)
2. [Pattern di Programmazione Ottimali](#pattern-programmazione)
3. [Comandi Shell Essenziali](#comandi-shell)
4. [Architetture di Riferimento](#architetture)
5. [Checklist Deployment](#checklist-deployment)
6. [Troubleshooting Comune](#troubleshooting)

---

# 1. PROGETTI COMPLETI PYTHON/JAVASCRIPT {#progetti-completi}

## 1.1 MyFileHub - Django File Sharing Platform
**Repository:** turiliffiu/myfilehub  
**Stack:** Django 5.1, PostgreSQL, Gunicorn, Nginx  
**Caratteristiche:**
- Sistema condivisione file con link pubblici/privati UUID-based
- Gestione permessi granulari (view, download, edit)
- Tracking accessi con IP e user agent
- Upload drag-and-drop con progress bar
- Interfaccia moderna con hover effects

**Codice chiave:**
```python
# models.py - Condivisione con UUID
class Share(models.Model):
    token = models.UUIDField(default=uuid.uuid4, unique=True)
    folder = models.ForeignKey(Folder, on_delete=models.CASCADE)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
    is_public = models.BooleanField(default=False)
    expires_at = models.DateTimeField(null=True, blank=True)
    permission_level = models.CharField(max_length=10, choices=[
        ('view', 'View Only'),
        ('download', 'Download'),
        ('edit', 'Edit')
    ])

# views.py - Tracking accessi
class ShareAccess(models.Model):
    share = models.ForeignKey(Share, on_delete=models.CASCADE)
    accessed_at = models.DateTimeField(auto_now_add=True)
    ip_address = models.GenericIPAddressField()
    user_agent = models.TextField()
    user = models.ForeignKey(User, null=True, on_delete=models.SET_NULL)
```

**Deployment:**
```bash
# Script installazione completo
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl restart gunicorn
sudo systemctl reload nginx
```

---

## 1.2 Dashboard Node.js - Procedure Operative
**Stack:** Node.js 20.x, Express 4.x, React 18.x, PostgreSQL 16.x  
**Caratteristiche:**
- Migrazione completa da Django a stack moderno
- 18 endpoints REST API con JWT authentication
- Sistema permessi 3-tier (admin/editor/viewer)
- Upload file con multer
- Ricerca full-text PostgreSQL
- Frontend React con Vite

**Codice chiave:**
```javascript
// Backend - JWT Middleware
const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'No token provided' });
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Sequelize ORM - Modello Procedura
const Procedure = sequelize.define('Procedure', {
  title: { type: DataTypes.STRING, allowNull: false },
  content: { type: DataTypes.TEXT },
  filename: { type: DataTypes.STRING, unique: true },
  is_public: { type: DataTypes.BOOLEAN, defaultValue: false },
  owner_id: {
    type: DataTypes.INTEGER,
    references: { model: 'Users', key: 'id' }
  }
});

// Frontend React - Upload con stato
const [uploadProgress, setUploadProgress] = useState(0);

const handleUpload = async (file) => {
  const formData = new FormData();
  formData.append('file', file);
  
  const config = {
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: (progressEvent) => {
      const percentCompleted = Math.round(
        (progressEvent.loaded * 100) / progressEvent.total
      );
      setUploadProgress(percentCompleted);
    }
  };
  
  await axios.post('/api/upload', formData, config);
};
```

---

## 1.3 AgriSecure - Sistema IoT Agricolo
**Stack:** Django, MQTT, Redis, WebSocket, ESP32-C6  
**Caratteristiche:**
- Mesh network 25 nodi ESP32
- Real-time con WebSocket e Redis
- MQTT-to-WebSocket bridge custom
- Discriminazione persona/animale con AI
- Dashboard live con aggiornamenti automatici

**Codice chiave:**
```python
# MQTT Consumer con Redis
import paho.mqtt.client as mqtt
import redis
import json

class MQTTToWebSocketBridge:
    def __init__(self):
        self.mqtt_client = mqtt.Client()
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
        
    def on_message(self, client, userdata, msg):
        """Riceve da MQTT e pubblica su Redis per WebSocket"""
        topic = msg.topic
        payload = json.loads(msg.payload.decode())
        
        # Pubblica su canale Redis per WebSocket consumers
        self.redis_client.publish(
            f'agrisecure:{topic}',
            json.dumps(payload)
        )
    
    def start(self):
        self.mqtt_client.on_message = self.on_message
        self.mqtt_client.connect('localhost', 1883, 60)
        self.mqtt_client.subscribe('agrisecure/+/+/#')
        self.mqtt_client.loop_forever()

# Django Channels WebSocket Consumer
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class AgriSecureConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        await self.accept()
        # Subscribe to Redis channel
        self.channel_layer.group_add('agrisecure_updates', self.channel_name)
    
    async def receive_json(self, content):
        # Handle incoming WebSocket messages
        message_type = content.get('type')
        if message_type == 'subscribe_node':
            node_id = content.get('node_id')
            await self.subscribe_to_node(node_id)
    
    async def agrisecure_update(self, event):
        # Send update to WebSocket
        await self.send_json(event['data'])

# Fix Decimal serialization per JSON
from decimal import Decimal
import json

class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

# Usage
data = {'temperature': Decimal('24.5'), 'humidity': Decimal('60.2')}
json_string = json.dumps(data, cls=DecimalEncoder)
```

**ESP32 Firmware Patterns:**
```cpp
// mesh_network.h - ESP-NOW Mesh Communication
struct MeshMessage {
    uint8_t sender_id;
    uint8_t msg_type;
    float sensor_value;
    uint32_t timestamp;
};

void initMeshNetwork() {
    WiFi.mode(WIFI_STA);
    esp_now_init();
    esp_now_register_recv_cb(onDataRecv);
    esp_now_register_send_cb(onDataSent);
}

void onDataRecv(const uint8_t *mac, const uint8_t *data, int len) {
    MeshMessage msg;
    memcpy(&msg, data, sizeof(msg));
    
    // Forward to gateway if not destination
    if (msg.sender_id != NODE_ID) {
        forwardToGateway(&msg);
    }
}
```

---

## 1.4 Talent Mosaic - Enterprise HR Management
**Stack:** Django 5.0, PostgreSQL 16+, Redis 7+, Celery  
**Caratteristiche:**
- 70 file Python, ~15,000 righe codice
- Admin dashboard completo
- AI matching algorithm per competenze
- Sistema badge e gamification
- Analytics e reporting avanzati
- Deployment Proxmox con automation

**Codice chiave:**
```python
# models.py - Skill Matching
from django.db import models

class UserSkill(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    skill = models.ForeignKey(Skill, on_delete=models.CASCADE)
    proficiency = models.IntegerField(choices=[
        (1, 'Beginner'), (2, 'Intermediate'),
        (3, 'Advanced'), (4, 'Expert')
    ])
    verified = models.BooleanField(default=False)
    verified_by = models.ForeignKey(User, null=True, on_delete=models.SET_NULL, related_name='verified_skills')

# AI Matching Algorithm
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def calculate_skill_match(user1, user2):
    """Calcola similarità competenze tra due utenti"""
    skills1 = user1.userskill_set.values_list('skill__name', 'proficiency')
    skills2 = user2.userskill_set.values_list('skill__name', 'proficiency')
    
    # Create skill vectors
    vectorizer = TfidfVectorizer()
    vectors = vectorizer.fit_transform([
        ' '.join([f"{s[0]}:{s[1]}" for s in skills1]),
        ' '.join([f"{s[0]}:{s[1]}" for s in skills2])
    ])
    
    similarity = cosine_similarity(vectors[0:1], vectors[1:2])[0][0]
    return round(similarity * 100, 2)

# Celery Task per notifiche
from celery import shared_task

@shared_task
def send_skill_match_notification(user_id, match_id):
    user = User.objects.get(id=user_id)
    match = User.objects.get(id=match_id)
    similarity = calculate_skill_match(user, match)
    
    # Send email notification
    send_mail(
        subject=f'New Skill Match: {match.get_full_name()}',
        message=f'You have a {similarity}% skill match with {match.get_full_name()}',
        from_email='noreply@talentmosaic.com',
        recipient_list=[user.email]
    )
```

---

## 1.5 Dashboard Django Procedure Operative
**Stack:** Django, PostgreSQL, JavaScript vanilla  
**Caratteristiche:**
- Sistema multi-utente con role-based permissions
- Upload e parsing file TXT strutturati
- Copy-paste comandi con un click
- Search e filtering real-time
- Gestione categorie CRUD completo

**Codice chiave:**
```python
# Parser personalizzato file TXT
def parse_procedure_file(file_path):
    """Parse structured text file format"""
    with open(file_path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    sections = []
    current_section = None
    
    for line in content.split('\n'):
        if line.startswith('[') and line.endswith(']'):
            if current_section:
                sections.append(current_section)
            current_section = {
                'title': line.strip('[]'),
                'commands': []
            }
        elif line.startswith('COMANDO:') and current_section:
            command = line.replace('COMANDO:', '').strip()
            current_section['commands'].append(command)
    
    if current_section:
        sections.append(current_section)
    
    return sections

# JavaScript - Copy to clipboard con fallback
function copyToClipboard(text, elementId) {
    if (navigator.clipboard) {
        navigator.clipboard.writeText(text).then(() => {
            showCopyFeedback(elementId);
        });
    } else {
        // Fallback per browser senza Clipboard API
        const textarea = document.createElement('textarea');
        textarea.value = text;
        document.body.appendChild(textarea);
        textarea.select();
        document.execCommand('copy');
        document.body.removeChild(textarea);
        showCopyFeedback(elementId);
    }
}

// Fix ID unici per elementi ripetuti
sections.forEach((section, sectionIndex) => {
    section.commands.forEach((command, commandIndex) => {
        const uniqueId = `cmd-${sectionIndex}-${commandIndex}`;
        // Use uniqueId for element IDs to avoid duplicates
    });
});
```

---

# 2. PATTERN DI PROGRAMMAZIONE OTTIMALI {#pattern-programmazione}

## 2.1 Django Best Practices

### Struttura Progetto Standard
```
myproject/
├── manage.py
├── requirements.txt
├── .env
├── .gitignore
├── myproject/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── apps/
│   ├── core/
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── serializers.py
│   │   └── permissions.py
│   └── api/
├── static/
├── media/
└── templates/
```

### Settings.py - Configurazione Sicura
```python
# settings.py - ENVIRONMENT VARIABLES
from pathlib import Path
import os
from dotenv import load_dotenv

load_dotenv()

# SECURITY
SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = os.getenv('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '').split(',')

# DATABASE
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

# STATIC & MEDIA FILES
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# CORS (se API)
CORS_ALLOWED_ORIGINS = os.getenv('CORS_ORIGINS', '').split(',')

# CELERY
CELERY_BROKER_URL = os.getenv('REDIS_URL', 'redis://localhost:6379/0')
CELERY_RESULT_BACKEND = os.getenv('REDIS_URL', 'redis://localhost:6379/0')
```

### Models - Best Practices
```python
from django.db import models
from django.contrib.auth.models import User
import uuid

class BaseModel(models.Model):
    """Abstract base con campi comuni"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class MyModel(BaseModel):
    # UUID per ID pubblici
    uuid = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)
    
    # ForeignKey con CASCADE appropriato
    owner = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='owned_items'
    )
    
    # Choices as tuples
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
        ('archived', 'Archived')
    ]
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='draft')
    
    # Indexes per performance
    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['owner', '-created_at']),
        ]
        ordering = ['-created_at']
    
    def __str__(self):
        return f"{self.uuid} - {self.status}"
```

### Views - Function-Based vs Class-Based
```python
# Function-Based View con decorators
from django.contrib.auth.decorators import login_required
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods

@login_required
@require_http_methods(["GET", "POST"])
def my_view(request):
    if request.method == 'POST':
        # Handle POST
        data = json.loads(request.body)
        # Process data
        return JsonResponse({'status': 'success'})
    else:
        # Handle GET
        return render(request, 'template.html', context)

# Class-Based View (preferito per CRUD)
from django.views.generic import ListView, DetailView, CreateView

class MyModelListView(ListView):
    model = MyModel
    template_name = 'mymodel_list.html'
    context_object_name = 'items'
    paginate_by = 20
    
    def get_queryset(self):
        # Ottimizzazione query con select_related
        return MyModel.objects.select_related('owner').filter(
            status='published'
        )

# REST API con Django REST Framework
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated

class MyModelViewSet(viewsets.ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        # Filtra per owner
        return self.queryset.filter(owner=self.request.user)
```

### Custom Permissions
```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        # Read permissions per tutti
        if request.method in permissions.SAFE_METHODS:
            return True
        # Write solo per owner
        return obj.owner == request.user

class IsAdminOrEditor(permissions.BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and (
            request.user.is_staff or 
            request.user.groups.filter(name='editors').exists()
        )
```

---

## 2.2 Node.js/Express Best Practices

### Struttura Progetto
```
backend/
├── src/
│   ├── config/
│   │   ├── database.js
│   │   └── config.js
│   ├── models/
│   │   └── *.model.js
│   ├── controllers/
│   │   └── *.controller.js
│   ├── routes/
│   │   └── *.routes.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── errorHandler.js
│   │   └── validation.js
│   ├── services/
│   │   └── *.service.js
│   ├── utils/
│   │   └── *.util.js
│   └── app.js
├── package.json
├── .env
└── server.js
```

### Server Setup con Best Practices
```javascript
// server.js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const { sequelize } = require('./src/config/database');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
    origin: process.env.CORS_ORIGIN,
    credentials: true
}));

// Logging
app.use(morgan('combined'));

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Routes
app.use('/api/auth', require('./src/routes/auth.routes'));
app.use('/api/users', require('./src/routes/user.routes'));

// Error handling middleware (must be last)
app.use(require('./src/middleware/errorHandler'));

// Database connection
sequelize.authenticate()
    .then(() => console.log('Database connected'))
    .catch(err => console.error('Database connection error:', err));

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

### JWT Authentication Middleware
```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = async (req, res, next) => {
    try {
        // Extract token
        const token = req.header('Authorization')?.replace('Bearer ', '');
        
        if (!token) {
            return res.status(401).json({ error: 'Authentication required' });
        }
        
        // Verify token
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        
        // Attach user to request
        req.user = decoded;
        req.userId = decoded.id;
        
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid or expired token' });
    }
};

// Role-based authorization
const requireRole = (roles) => {
    return (req, res, next) => {
        if (!roles.includes(req.user.role)) {
            return res.status(403).json({ error: 'Insufficient permissions' });
        }
        next();
    };
};

module.exports = { authMiddleware, requireRole };
```

### Sequelize Models Best Practices
```javascript
// models/User.model.js
const { DataTypes } = require('sequelize');
const bcrypt = require('bcrypt');

module.exports = (sequelize) => {
    const User = sequelize.define('User', {
        id: {
            type: DataTypes.INTEGER,
            primaryKey: true,
            autoIncrement: true
        },
        email: {
            type: DataTypes.STRING,
            unique: true,
            allowNull: false,
            validate: {
                isEmail: true
            }
        },
        password: {
            type: DataTypes.STRING,
            allowNull: false
        },
        role: {
            type: DataTypes.ENUM('admin', 'editor', 'viewer'),
            defaultValue: 'viewer'
        }
    }, {
        hooks: {
            beforeCreate: async (user) => {
                user.password = await bcrypt.hash(user.password, 10);
            },
            beforeUpdate: async (user) => {
                if (user.changed('password')) {
                    user.password = await bcrypt.hash(user.password, 10);
                }
            }
        },
        defaultScope: {
            attributes: { exclude: ['password'] }
        },
        scopes: {
            withPassword: {
                attributes: { include: ['password'] }
            }
        }
    });
    
    // Instance methods
    User.prototype.comparePassword = async function(password) {
        return bcrypt.compare(password, this.password);
    };
    
    return User;
};
```

### Error Handling Middleware
```javascript
// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
    console.error('Error:', err);
    
    // Sequelize validation errors
    if (err.name === 'SequelizeValidationError') {
        return res.status(400).json({
            error: 'Validation error',
            details: err.errors.map(e => ({
                field: e.path,
                message: e.message
            }))
        });
    }
    
    // Sequelize unique constraint
    if (err.name === 'SequelizeUniqueConstraintError') {
        return res.status(409).json({
            error: 'Resource already exists',
            field: err.errors[0].path
        });
    }
    
    // JWT errors
    if (err.name === 'JsonWebTokenError') {
        return res.status(401).json({ error: 'Invalid token' });
    }
    
    // Default error
    res.status(err.status || 500).json({
        error: err.message || 'Internal server error'
    });
};

module.exports = errorHandler;
```

---

## 2.3 React Best Practices

### Component Structure
```javascript
// components/UserProfile.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserProfile = ({ userId }) => {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    
    useEffect(() => {
        const fetchUser = async () => {
            try {
                setLoading(true);
                const response = await axios.get(`/api/users/${userId}`, {
                    headers: {
                        'Authorization': `Bearer ${localStorage.getItem('token')}`
                    }
                });
                setUser(response.data);
            } catch (err) {
                setError(err.message);
            } finally {
                setLoading(false);
            }
        };
        
        fetchUser();
    }, [userId]);
    
    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;
    if (!user) return null;
    
    return (
        <div className="user-profile">
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
};

export default UserProfile;
```

### Custom Hooks
```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        const token = localStorage.getItem('token');
        if (token) {
            axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
            fetchCurrentUser();
        } else {
            setLoading(false);
        }
    }, []);
    
    const fetchCurrentUser = async () => {
        try {
            const response = await axios.get('/api/auth/me');
            setUser(response.data);
        } catch (error) {
            localStorage.removeItem('token');
        } finally {
            setLoading(false);
        }
    };
    
    const login = async (email, password) => {
        const response = await axios.post('/api/auth/login', { email, password });
        localStorage.setItem('token', response.data.token);
        axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
        setUser(response.data.user);
    };
    
    const logout = () => {
        localStorage.removeItem('token');
        delete axios.defaults.headers.common['Authorization'];
        setUser(null);
    };
    
    return { user, loading, login, logout };
};
```

---

# 3. COMANDI SHELL ESSENZIALI {#comandi-shell}

## 3.1 Git & Version Control

```bash
# Setup iniziale repository
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/username/repo.git
git push -u origin main

# Workflow quotidiano
git status                          # Check modifiche
git add .                           # Stage tutte le modifiche
git commit -m "Descrizione chiara"  # Commit con messaggio
git push origin main                # Push su remote

# Branch management
git checkout -b feature/new-feature # Crea e switcha a nuovo branch
git checkout main                   # Torna a main
git merge feature/new-feature       # Merge branch in main
git branch -d feature/new-feature   # Elimina branch locale

# Undo operations
git reset --hard HEAD~1             # Undo ultimo commit (PERICOLOSO)
git reset --soft HEAD~1             # Undo commit ma mantieni modifiche
git checkout -- filename            # Scarta modifiche file specifico
git clean -fd                       # Rimuovi file untracked

# Stash - salva modifiche temporanee
git stash                           # Salva modifiche
git stash list                      # Lista stash
git stash apply                     # Applica ultimo stash
git stash pop                       # Applica e rimuovi ultimo stash

# Remote management
git remote -v                       # Lista remote
git remote remove origin            # Rimuovi remote
git fetch origin                    # Scarica modifiche senza merge
git pull origin main                # Scarica e merge
```

## 3.2 Django Development

```bash
# Setup progetto Django
python -m venv venv                        # Crea virtual environment
source venv/bin/activate                   # Attiva venv (Linux/Mac)
venv\Scripts\activate                      # Attiva venv (Windows)

pip install django djangorestframework    # Installa dipendenze base
pip install psycopg2-binary python-dotenv # Database e env vars
pip freeze > requirements.txt              # Salva dipendenze

django-admin startproject myproject .      # Crea progetto
python manage.py startapp myapp            # Crea app

# Database migrations
python manage.py makemigrations            # Crea migrations
python manage.py makemigrations myapp      # Migrations per app specifica
python manage.py migrate                   # Applica migrations
python manage.py showmigrations            # Lista migrations
python manage.py migrate myapp zero        # Undo tutte migrations app
python manage.py migrate --fake myapp 0001 # Fake migration

# Superuser e shell
python manage.py createsuperuser           # Crea admin user
python manage.py shell                     # Django shell interattiva
python manage.py shell_plus                # Shell avanzata (django-extensions)

# Static files
python manage.py collectstatic             # Raccoglie static files
python manage.py collectstatic --noinput   # Senza conferma (per script)

# Development server
python manage.py runserver                 # Default 127.0.0.1:8000
python manage.py runserver 0.0.0.0:8000   # Accessibile da rete

# Database utilities
python manage.py dbshell                   # SQL shell diretta
python manage.py dumpdata > data.json      # Export dati
python manage.py loaddata data.json        # Import dati
python manage.py flush                     # Pulisci database (DANGER!)

# Custom management commands
python manage.py populate_db               # Comando custom per seed data
```

## 3.3 Node.js/npm

```bash
# Setup progetto Node.js
npm init -y                               # Crea package.json
npm install express sequelize pg          # Installa dipendenze
npm install --save-dev nodemon            # Dev dependencies

# Package management
npm install package_name                  # Installa package
npm install package_name@version          # Versione specifica
npm install                               # Installa da package.json
npm update                                # Aggiorna packages
npm uninstall package_name                # Rimuovi package
npm list                                  # Lista packages installati
npm list --depth=0                        # Solo top-level packages

# Scripts in package.json
npm run dev                               # Script "dev" definito in package.json
npm start                                 # Script "start"
npm test                                  # Script "test"

# Esempio package.json scripts
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "migrate": "sequelize-cli db:migrate",
    "seed": "sequelize-cli db:seed:all"
  }
}

# Cache management
npm cache clean --force                   # Pulisci cache npm
rm -rf node_modules package-lock.json     # Reset completo
npm install                               # Reinstalla tutto
```

## 3.4 PostgreSQL

```bash
# Accesso PostgreSQL
sudo -u postgres psql                     # Login come postgres user
psql -U username -d database_name         # Login specifico

# Database management
CREATE DATABASE mydb;                     # Crea database
DROP DATABASE mydb;                       # Elimina database
\l                                        # Lista databases
\c mydb                                   # Connetti a database

# User management
CREATE USER myuser WITH PASSWORD 'pass';  # Crea user
ALTER USER myuser WITH PASSWORD 'newpass';# Cambia password
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
\du                                       # Lista users

# Table operations
\dt                                       # Lista tabelle
\d table_name                             # Descrivi tabella
SELECT * FROM table_name LIMIT 10;        # Query sample data

# Backup e restore
pg_dump -U username dbname > backup.sql   # Backup
psql -U username dbname < backup.sql      # Restore

# Performance
VACUUM ANALYZE;                           # Ottimizza database
REINDEX DATABASE dbname;                  # Rebuild indexes
```

## 3.5 Deployment con Gunicorn & Nginx

```bash
# Installazione Gunicorn
pip install gunicorn

# Test Gunicorn manuale
gunicorn myproject.wsgi:application --bind 0.0.0.0:8000

# Systemd service file: /etc/systemd/system/gunicorn.service
[Unit]
Description=gunicorn daemon for myproject
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/myproject
Environment="PATH=/opt/myproject/venv/bin"
ExecStart=/opt/myproject/venv/bin/gunicorn \
    --workers 3 \
    --bind unix:/opt/myproject/gunicorn.sock \
    myproject.wsgi:application

[Install]
WantedBy=multi-user.target

# Gestione servizio
sudo systemctl daemon-reload              # Ricarica configurazione
sudo systemctl start gunicorn             # Avvia servizio
sudo systemctl enable gunicorn            # Avvio automatico al boot
sudo systemctl status gunicorn            # Check status
sudo systemctl restart gunicorn           # Restart
sudo systemctl stop gunicorn              # Stop

# Logs
sudo journalctl -u gunicorn               # Tutti i logs
sudo journalctl -u gunicorn -f            # Follow logs real-time
sudo journalctl -u gunicorn --since today # Logs di oggi

# Nginx configuration: /etc/nginx/sites-available/myproject
server {
    listen 80;
    server_name example.com;

    location /static/ {
        alias /opt/myproject/staticfiles/;
    }

    location /media/ {
        alias /opt/myproject/media/;
    }

    location / {
        proxy_pass http://unix:/opt/myproject/gunicorn.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Nginx commands
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo nginx -t                             # Test configuration
sudo systemctl restart nginx              # Restart nginx
sudo systemctl reload nginx               # Reload config
sudo tail -f /var/log/nginx/error.log     # Error logs
sudo tail -f /var/log/nginx/access.log    # Access logs
```

## 3.6 File Permissions & Ownership

```bash
# Ownership Django project
sudo chown -R www-data:www-data /opt/myproject
sudo chown -R www-data:www-data /opt/myproject/media
sudo chown -R www-data:www-data /opt/myproject/staticfiles

# Permissions
sudo chmod -R 755 /opt/myproject
sudo chmod -R 775 /opt/myproject/media      # Writable
sudo chmod 600 /opt/myproject/.env          # Solo owner read/write

# Find files con permessi sbagliati
find /opt/myproject -type f -not -user www-data
find /opt/myproject -type d -not -perm 755

# Fix batch permissions
find /opt/myproject -type f -exec chmod 644 {} \;
find /opt/myproject -type d -exec chmod 755 {} \;
```

## 3.7 Process Management con PM2 (Node.js)

```bash
# Installazione PM2
npm install -g pm2

# Avvio applicazione
pm2 start server.js --name myapp          # Start con nome
pm2 start server.js -i 4                  # Cluster mode 4 istanze
pm2 start npm -- start                    # Start npm script

# Management
pm2 list                                  # Lista processi
pm2 status                                # Status processi
pm2 restart myapp                         # Restart
pm2 stop myapp                            # Stop
pm2 delete myapp                          # Delete
pm2 reload myapp                          # Zero-downtime reload

# Logs
pm2 logs                                  # Tutti i logs
pm2 logs myapp                            # Logs app specifica
pm2 logs myapp --lines 100                # Ultime 100 righe
pm2 flush                                 # Pulisci logs

# Monitoring
pm2 monit                                 # Dashboard real-time
pm2 status                                # Quick status

# Startup script (avvio automatico boot)
pm2 startup                               # Genera comando startup
pm2 save                                  # Salva configurazione attuale
```

## 3.8 Docker (Bonus)

```bash
# Build e run
docker build -t myapp .                   # Build image
docker run -p 8000:8000 myapp             # Run container
docker run -d --name myapp-container myapp # Run in background

# Management
docker ps                                 # Running containers
docker ps -a                              # Tutti i containers
docker stop container_id                  # Stop container
docker rm container_id                    # Remove container
docker images                             # Lista images
docker rmi image_id                       # Remove image

# Logs e exec
docker logs container_id                  # View logs
docker logs -f container_id               # Follow logs
docker exec -it container_id /bin/bash    # Shell interattiva

# Docker Compose
docker-compose up -d                      # Start services
docker-compose down                       # Stop e remove
docker-compose logs -f service_name       # Logs servizio
docker-compose restart service_name       # Restart servizio
```

## 3.9 System Monitoring

```bash
# Process monitoring
top                                       # Monitor processi
htop                                      # Top migliorato (installare)
ps aux | grep gunicorn                    # Trova processo specifico
kill -9 PID                               # Kill processo

# Disk usage
df -h                                     # Spazio disco
du -sh /opt/myproject/*                   # Dimensione directory
du -h --max-depth=1 /opt                  # Size subdirectories

# Memory
free -h                                   # RAM usage
cat /proc/meminfo                         # Dettagli memoria

# Network
netstat -tuln                             # Porte in ascolto
lsof -i :8000                             # Chi usa porta 8000
curl -I http://localhost:8000             # Test HTTP

# System logs
sudo tail -f /var/log/syslog              # System log
sudo journalctl -xe                       # Systemd logs
sudo dmesg | tail                         # Kernel messages
```

---

# 4. ARCHITETTURE DI RIFERIMENTO {#architetture}

## 4.1 Stack Django Production-Ready

```
┌─────────────────────────────────────────────────────┐
│                    NGINX (Reverse Proxy)            │
│            ┌────────────┐     ┌────────────┐        │
│            │  Static    │     │   Media    │        │
│            │   Files    │     │   Files    │        │
│            └────────────┘     └────────────┘        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Gunicorn (WSGI Server)                 │
│         ┌──────────┬──────────┬──────────┐          │
│         │ Worker 1 │ Worker 2 │ Worker 3 │          │
│         └──────────┴──────────┴──────────┘          │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Django Application                     │
│  ┌───────────┐  ┌──────────┐  ┌──────────────┐     │
│  │   Views   │  │  Models  │  │  Middleware  │     │
│  └───────────┘  └──────────┘  └──────────────┘     │
└──────┬────────────────┬────────────────┬───────────┘
       │                │                │
┌──────▼────────┐ ┌────▼─────┐ ┌────────▼──────────┐
│  PostgreSQL   │ │  Redis   │ │  Celery Workers  │
│   Database    │ │  Cache   │ │  (Background)     │
└───────────────┘ └──────────┘ └──────────────────┘
```

**Componenti:**
- **Nginx:** Reverse proxy, serve static/media, SSL termination
- **Gunicorn:** WSGI server con worker multipli
- **Django:** Application logic
- **PostgreSQL:** Database relazionale
- **Redis:** Caching, Celery broker
- **Celery:** Task asincroni (email, processing)

---

## 4.2 Stack Node.js + React Production

```
┌─────────────────────────────────────────────────────┐
│                    NGINX                            │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │   React Build    │      │   API Proxy      │    │
│  │  (Static Files)  │      │  /api/* → 3000   │    │
│  └──────────────────┘      └──────────────────┘    │
└────────────────────────────────┬───────────────────┘
                                 │
            ┌────────────────────┴────────────────┐
            │                                     │
┌───────────▼────────┐              ┌────────────▼──────┐
│   React Frontend   │              │  Express Backend  │
│   (Port 80/443)    │              │    (Port 3000)    │
│                    │◄─────API─────┤                   │
│  - Vite/Webpack    │              │  - REST API       │
│  - Tailwind CSS    │              │  - JWT Auth       │
│  - React Router    │              │  - Sequelize ORM  │
└────────────────────┘              └────────┬──────────┘
                                             │
                             ┌───────────────┴──────────────┐
                             │                              │
                    ┌────────▼────────┐          ┌─────────▼────────┐
                    │   PostgreSQL    │          │      Redis       │
                    │   (Database)    │          │  (Sessions/Cache)│
                    └─────────────────┘          └──────────────────┘
```

**Deployment:**
- Frontend React buildato e servito come static files da Nginx
- Backend Node.js con PM2 cluster mode
- Nginx fa reverse proxy delle API calls a Express
- PostgreSQL per dati persistenti
- Redis per sessioni e caching

---

## 4.3 Microservices con IoT (AgriSecure Style)

```
┌────────────────────────────────────────────────────────────┐
│                        ESP32 Mesh Network                   │
│  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐     │
│  │Node 1│───│Node 2│───│Node 3│───│Node 4│───│Node 5│     │
│  │(Sens)│   │(Sens)│   │(Gate)│   │(Sens)│   │(Sens)│     │
│  └──────┘   └──────┘   └──┬───┘   └──────┘   └──────┘     │
└──────────────────────────│─────────────────────────────────┘
                           │ 4G/WiFi
┌──────────────────────────▼─────────────────────────────────┐
│              MQTT Broker (Mosquitto)                       │
│              Topic: agrisecure/+/+/#                       │
└──────────────────────────┬─────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│            MQTT-to-WebSocket Bridge                        │
│  ┌─────────────────┐           ┌──────────────────┐        │
│  │  MQTT Consumer  │───────────►  Redis Pub/Sub   │        │
│  │  (Python/Paho)  │           │  (Channel Layer) │        │
│  └─────────────────┘           └──────────────────┘        │
└──────────────────────────┬─────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│                Django Application                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐        │
│  │  REST API  │  │   Models   │  │   WebSocket    │        │
│  │  (DRF)     │  │ (Postgres) │  │  (Channels)    │        │
│  └────────────┘  └────────────┘  └────────────────┘        │
└──────────────────────┬──────────────────┬──────────────────┘
                       │                  │
        ┌──────────────▼──────┐  ┌────────▼──────────┐
        │    PostgreSQL       │  │   Celery Workers  │
        │  - Sensor Data      │  │  - Email Alerts   │
        │  - Events Log       │  │  - SMS/Telegram   │
        │  - User Settings    │  │  - Data Analysis  │
        └─────────────────────┘  └───────────────────┘
                                          │
                                 ┌────────▼────────┐
                                 │   Redis Broker  │
                                 └─────────────────┘
```

**Flusso dati:**
1. ESP32 nodes → MQTT broker
2. MQTT consumer Python → Redis Pub/Sub
3. Django Channels WebSocket → Real-time updates browser
4. Celery tasks → Background processing alerts
5. REST API → CRUD operazioni dashboard

---

# 5. CHECKLIST DEPLOYMENT {#checklist-deployment}

## 5.1 Django Deployment Checklist

```bash
# 1. ENVIRONMENT SETUP
[ ] Crea virtual environment
[ ] Installa dipendenze: pip install -r requirements.txt
[ ] Configura variabili .env (SECRET_KEY, DB, etc.)
[ ] DEBUG = False in settings.py
[ ] ALLOWED_HOSTS configurato correttamente

# 2. DATABASE
[ ] Crea database PostgreSQL
[ ] Crea user database con password strong
[ ] Grant privilegi al user
[ ] python manage.py migrate
[ ] python manage.py createsuperuser

# 3. STATIC & MEDIA FILES
[ ] STATIC_ROOT configurato
[ ] STATICFILES_DIRS configurato se necessario
[ ] python manage.py collectstatic
[ ] Permessi corretti su /media e /static
[ ] sudo chown -R www-data:www-data media/
[ ] sudo chown -R www-data:www-data staticfiles/

# 4. GUNICORN
[ ] pip install gunicorn
[ ] Test gunicorn manualmente
[ ] Crea /etc/systemd/system/gunicorn.service
[ ] sudo systemctl daemon-reload
[ ] sudo systemctl start gunicorn
[ ] sudo systemctl enable gunicorn
[ ] Check logs: sudo journalctl -u gunicorn

# 5. NGINX
[ ] Installa nginx: sudo apt install nginx
[ ] Crea /etc/nginx/sites-available/myproject
[ ] Symlink a sites-enabled
[ ] Test config: sudo nginx -t
[ ] sudo systemctl restart nginx
[ ] Check logs: /var/log/nginx/error.log

# 6. SECURITY
[ ] SSL certificate (Let's Encrypt)
[ ] CSRF_TRUSTED_ORIGINS configurato
[ ] SECURE_SSL_REDIRECT = True (se HTTPS)
[ ] SESSION_COOKIE_SECURE = True
[ ] CSRF_COOKIE_SECURE = True
[ ] Firewall configurato (UFW)

# 7. MONITORING
[ ] Setup logging in settings.py
[ ] Configura log rotation
[ ] Monitora spazio disco
[ ] Setup backup database automatico

# 8. FINAL CHECKS
[ ] Test tutti gli endpoints
[ ] Test file upload
[ ] Test email sending (se presente)
[ ] Test Celery tasks (se presente)
[ ] Verifica performance con wrk/ab
```

---

## 5.2 Node.js Deployment Checklist

```bash
# 1. ENVIRONMENT
[ ] Node.js 20.x installato
[ ] npm install (production)
[ ] File .env configurato
[ ] NODE_ENV=production

# 2. DATABASE
[ ] PostgreSQL installato e configurato
[ ] Database creato
[ ] User con permessi corretti
[ ] npx sequelize-cli db:migrate
[ ] npx sequelize-cli db:seed:all (se seed data)

# 3. PM2 SETUP
[ ] npm install -g pm2
[ ] pm2 start server.js --name myapp -i max
[ ] pm2 startup (auto-start)
[ ] pm2 save
[ ] Test restart: pm2 restart myapp

# 4. NGINX REVERSE PROXY
[ ] Nginx installato
[ ] Config file creato
[ ] Proxy_pass configurato
[ ] CORS headers se necessario
[ ] sudo systemctl restart nginx

# 5. SECURITY
[ ] Helmet.js configurato
[ ] Rate limiting attivo
[ ] JWT_SECRET strong e sicuro
[ ] CORS origins restrittivi
[ ] Input validation su tutte le route

# 6. MONITORING
[ ] pm2 logs configurato
[ ] Log rotation attivo
[ ] pm2 monit per monitoring
[ ] Alerts configurati

# 7. FINAL TESTS
[ ] Test tutte le API endpoints
[ ] Test autenticazione e autorizzazione
[ ] Test file upload
[ ] Load testing
[ ] Check memory leaks
```

---

# 6. TROUBLESHOOTING COMUNE {#troubleshooting}

## 6.1 Django Issues

### 403 Forbidden su Static Files
```bash
# Problema: Nginx non può accedere a static files
# Soluzione:
sudo chown -R www-data:www-data /opt/myproject/staticfiles
sudo chmod -R 755 /opt/myproject/staticfiles
# Verifica config Nginx: location /static/ deve puntare a STATIC_ROOT
```

### 502 Bad Gateway
```bash
# Problema: Gunicorn non risponde
# Debug:
sudo systemctl status gunicorn
sudo journalctl -u gunicorn -n 50
# Verifica socket:
ls -la /opt/myproject/gunicorn.sock
# Riavvia:
sudo systemctl restart gunicorn
```

### Migration Errors
```bash
# Problema: Migrations confusione o tabelle mancanti
# Reset migrations (DANGER - solo development):
python manage.py migrate myapp zero
rm myapp/migrations/000*.py
python manage.py makemigrations
python manage.py migrate

# Production - fake migration:
python manage.py migrate --fake myapp 0001_initial
```

### CORS Errors
```python
# settings.py
INSTALLED_APPS = [
    'corsheaders',
    # ...
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    # ...
]

CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://yourdomain.com",
]
```

---

## 6.2 Node.js/PostgreSQL Issues

### Connection Pool Errors
```javascript
// Aumenta pool size in Sequelize config
const sequelize = new Sequelize(process.env.DB_URL, {
    pool: {
        max: 10,
        min: 2,
        acquire: 30000,
        idle: 10000
    }
});
```

### JWT Token Issues
```javascript
// Verifica token expiration
const decoded = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ['HS256']
});

// Token refresh pattern
if (decoded.exp - Date.now()/1000 < 300) { // 5 min rimanenti
    const newToken = generateToken(user);
    res.setHeader('X-New-Token', newToken);
}
```

### Sequelize Query Performance
```javascript
// N+1 problem - usa include
const users = await User.findAll({
    include: [{ model: Profile }, { model: Posts }]
});

// Lazy loading invece di eager loading
const user = await User.findByPk(id);
const posts = await user.getPosts(); // Solo quando necessario
```

---

## 6.3 General System Issues

### Port Already in Use
```bash
# Trova processo su porta 8000
sudo lsof -i :8000
# Kill processo
sudo kill -9 PID
```

### Out of Disk Space
```bash
# Check spazio
df -h
# Trova directory grandi
du -sh /* | sort -rh | head -10
# Pulisci logs
sudo journalctl --vacuum-time=7d
# Pulisci package cache
sudo apt clean
sudo apt autoremove
```

### Memory Leaks Node.js
```bash
# Monitor memory
pm2 monit
# Heap snapshot
node --inspect server.js
# Chrome DevTools -> Memory tab
# Restart periodico
pm2 start server.js --max-memory-restart 1G
```

---

# CONCLUSIONI

Questo documento raccoglie i pattern e le best practices dai 20 progetti Python/JavaScript completati. Utilizzalo come riferimento per:

1. **Setup rapido** nuovi progetti con architetture testate
2. **Troubleshooting** problemi comuni già risolti
3. **Deployment** produzione sicuro e scalabile
4. **Code quality** mantenendo standard elevati

## Prossimi Passi Consigliati

1. ⭐ Crea template repository GitHub con struttura base Django/Node.js
2. 📝 Aggiungi script bash automation per setup comuni
3. 🔒 Implementa CI/CD con GitHub Actions
4. 📊 Setup monitoring con Prometheus + Grafana
5. 🐳 Dockerizza applicazioni per deployment consistente

---

**Documento creato:** 1 Febbraio 2026  
**Basato su:** 89 conversazioni, 20 progetti completi  
**Mantenere aggiornato:** Aggiungere nuovi pattern man mano che emergono
