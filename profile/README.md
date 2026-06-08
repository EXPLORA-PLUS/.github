<div align="center">

<img src="https://raw.githubusercontent.com/EXPLORA-PLUS/explora-plus-docs/main/paper/figuras/brasao-unisantos.png" alt="Universidade Católica de Santos" width="120" />

# Explora+

**Planejador de rotas turísticas pedestres com descoberta automática de POIs no OpenStreetMap**

Projeto acadêmico da Universidade Católica de Santos (UniSantos) — disciplina de Projeto de Conclusão (PCE).
Dado um par origem/destino, o Explora+ calcula a rota a pé via **OSRM (modo `foot`)**, descobre Pontos de Interesse (POIs) próximos via **Overpass API**, enriquece cada POI progressivamente com dados do **Nominatim, Wikidata e Wikipedia** e persiste tudo num catálogo canônico (`places.Place`) com geometria PostGIS. Cada usuário autenticado tem uma biblioteca pessoal (`UserPlaceState`) que rastreia o que já viu, visitou ou removeu de uma rota.

</div>

---

## Sumário

- [Repositórios](#repositórios)
- [Visão de produto](#visão-de-produto)
- [Arquitetura](#arquitetura)
- [Stack](#stack)
- [Como subir o ambiente completo](#como-subir-o-ambiente-completo)
- [Diagramas](#diagramas)
- [Colaboradores](#colaboradores)
- [Instituição](#instituição)

---

## Repositórios

O Explora+ é composto por **três repositórios** versionados sob a organização [`EXPLORA-PLUS`](https://github.com/EXPLORA-PLUS) no GitHub. A recomendação é cloná-los lado a lado em uma mesma pasta de workspace:

```text
explora_plus/
├── backend/    # API Django + PostGIS + planner de rotas (docker-compose mora aqui)
├── frontend/   # App Expo / React Native (web + mobile)
└── docs/       # SETUP, modelagem, diagramas UML, paper LaTeX
```

| Repositório | Papel | Stack principal | Link |
|---|---|---|---|
| **explora-plus-backend** | API REST, planner, autenticação JWT, persistência canônica de lugares e rotas | Django 5.1 · DRF · SimpleJWT · PostgreSQL + PostGIS · Python 3.12 | <https://github.com/EXPLORA-PLUS/explora-plus-backend> |
| **explora-plus-frontend** | App cliente (web e mobile), mapa Leaflet em WebView/iframe, biblioteca pessoal | Expo 52 · React Native 0.76 · TypeScript · React Navigation 7 | <https://github.com/EXPLORA-PLUS/explora-plus-frontend> |
| **explora-plus-docs** | Documentação técnica, modelagem, diagramas UML/Mermaid, pacote final do Modelio e papers acadêmicos | Markdown · Mermaid · Modelio · LaTeX | <https://github.com/EXPLORA-PLUS/explora-plus-docs> |

> Para clonar tudo de uma vez:
> ```bash
> mkdir explora_plus && cd explora_plus
> git clone https://github.com/EXPLORA-PLUS/explora-plus-backend.git  backend
> git clone https://github.com/EXPLORA-PLUS/explora-plus-frontend.git frontend
> git clone https://github.com/EXPLORA-PLUS/explora-plus-docs.git     docs
> ```

---

## Visão de produto

<div align="center">
<img src="https://raw.githubusercontent.com/EXPLORA-PLUS/.github/main/profile/assets/app-explorar.png" alt="Tela Explorar do Explora+ — mapa com rota turística, POIs numerados e resumo da rota" width="320" />
</div>

O Explora+ é uma **plataforma de propósito geral** para descoberta e navegação turística pedestre. Por ser construído inteiramente sobre dados abertos globais (OpenStreetMap, Wikidata, Wikipedia), o app não depende de parcerias comerciais locais e pode ser implantado em qualquer município com acervo registrado no OSM.

A entrega atual corresponde à **validação da arquitetura como MVP**: demonstra que o núcleo técnico (roteamento, enriquecimento progressivo de POIs, personalização por usuário) é sólido e escalável. As integrações com plataformas de eventos e mobilidade (Sympla, Uber etc.) e a definição da estratégia de distribuição e monetização constituem a próxima fase.

### Atores

A modelagem distingue dois atores ([UC1–UC12](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/MODELAGEM.md#casos-de-uso)):

- **Visitante (anônimo)** — pode visualizar o mapa com rota e POIs, filtrar por categoria, criar conta e fazer login.
- **Turista (autenticado, *estende* Visitante)** — gera rotas turísticas, abre detalhe de POI, marca/desmarca como visitado, exclui POIs da rota atual, vê biblioteca pessoal de lugares e perfil.

### Fluxo do app

1. **Bootstrap da sessão** — `AuthContext` tenta `POST /api/auth/refresh/` com o refresh token em storage; se falhar, cai em estado anônimo.
2. **Tela Explorar** — chama `GET /api/tour-routes/current/`. Se 404 (sem rota salva), dispara `POST /api/tour-routes/` com origem/destino padrão (Praça Oswaldo Cruz → Edifício Gibraltar, Av. Paulista).
3. **Planner do backend** — geocodifica via **Nominatim**, calcula rota pedestre via **OSRM**, busca POIs na bbox via **Overpass** e faz upsert em `places.Place`. Resultado entra em `RouteSearchCache` por chave canônica (origem+destino), evitando recálculos.
4. **Renderização** — `MapView` (Leaflet em `WebView` mobile / `iframe` web) desenha polyline + marcadores. POIs filtráveis por **Todos / Cultura / Parques / Comida**.
5. **Pré-busca em background** — após carregar a rota, o frontend pré-busca o detalhe de todos os POIs em background (400 ms entre chamadas), pra que abrir o modal seja instantâneo.
6. **Detalhe de POI** — `GET /api/tour-routes/pois/<stop_id>/` enriquece progressivamente: Nominatim (endereço, horários, `wikidata_id`) → Wikidata (imagem P18) → fallback de busca Wikipedia em `pt` → `en` se necessário.
7. **Personalização da rota** — cada `TourRouteStop` tem estado `active → visited → excluded` ([state machine](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/MODELAGEM.md#estados--stop-na-rota)). Ao marcar como visitado, o backend recalcula a rota sem o POI no trajeto ativo.
8. **Biblioteca pessoal** — aba **Lugares** lista `UserPlaceState` do usuário, com filtros: Todos / Visitados / Não visitados / Rota atual / Excluídos.

---

## Arquitetura

### Componentes

```mermaid
flowchart TB
    subgraph Frontend["Aplicativo (Expo + React Native)"]
        Screens["Telas\nExploreScreen / PlacesScreen / ProfileScreen / SearchSettingsScreen"]
        Auth["AuthContext\nJWT tokens"]
        Services["Services\napi.ts / tourRoutes.ts / auth.ts"]
        Map["MapView\nLeaflet via WebView/iframe"]
        Screens --> Auth
        Screens --> Services
        Screens --> Map
    end

    subgraph Backend["API Backend (Django 5.1 + DRF)"]
        AuthAPI["accounts\nPOST /api/auth/*\nGET /api/me/"]
        PlacesAPI["places\nGET /api/places/*"]
        TourRoutesAPI["tour_routes\nPOST /api/tour-routes/\nGET/PATCH /api/tour-routes/preferences/\nGET /api/tour-routes/current/\nGET /api/tour-routes/places/\nGET /api/tour-routes/pois/id/\nPATCH .../stops/id/state/\nDELETE .../stops/id/"]
        CoreAPI["core\nGET /api/health/"]
        TicketsAPI["tickets\nEndpoint mockado"]
    end

    DB[(PostgreSQL 16\n+ PostGIS 3.4)]

    subgraph Externos["APIs Externas (gratuitas/open)"]
        Nominatim["Nominatim\nGeocodificacao e extratags OSM"]
        OSRM["OSRM\nRoteamento pedestre"]
        Overpass["Overpass API\nBusca de POIs via OpenStreetMap"]
        Wikidata["Wikidata API\nImagem principal do lugar P18"]
        Wikipedia["Wikipedia REST API\nResumo e imagem fallback\npt e en"]
    end

    Services -->|HTTPS / JSON| AuthAPI
    Services -->|HTTPS / JSON| TourRoutesAPI
    Services -->|HTTPS / JSON| PlacesAPI

    AuthAPI --> DB
    PlacesAPI --> DB
    TourRoutesAPI --> DB
    TourRoutesAPI -->|geocoding| Nominatim
    TourRoutesAPI -->|rota pedestre| OSRM
    TourRoutesAPI -->|POIs filtrados por preferencias| Overpass
    TourRoutesAPI -->|imagem| Wikidata
    TourRoutesAPI -->|resumo e fallback| Wikipedia
```

### Implantação

```mermaid
flowchart TB
    subgraph Turista["Dispositivo do Turista"]
        App["App Expo Web / React Native\nhttp://localhost:8082"]
    end

    subgraph DevMachine["Maquina de Desenvolvimento (Docker Compose)"]
        subgraph DockerNet["Rede Docker Interna"]
            BackendC["Container: backend\nDjango 5.1 + Gunicorn\nporta interna 8000\nporta host 8080"]
            FrontendC["Container: frontend\nExpo Web\nporta interna 8081\nporta host 8082"]
            DBC["Container: db\npostgis/postgis 16-3.4\nporta interna 5432\nporta host 5433"]
            BackendC <--> DBC
        end
    end

    subgraph Externos["Provedores Externos"]
        Nominatim["Nominatim\nnominatim.openstreetmap.org"]
        OSRM["OSRM\nrouter.project-osrm.org"]
        Overpass["Overpass API\noverpass-api.de"]
        Wikidata["Wikidata\nwww.wikidata.org"]
        Wikipedia["Wikipedia\npt.wikipedia.org\nen.wikipedia.org"]
    end

    App -->|HTTPS / JSON porta 8080| BackendC
    BackendC -->|geocoding| Nominatim
    BackendC -->|roteamento| OSRM
    BackendC -->|POIs OSM| Overpass
    BackendC -->|imagem P18| Wikidata
    BackendC -->|resumo e busca| Wikipedia
```

### Fluxo de alto nível

```mermaid
flowchart LR
    FE[Frontend Expo/RN] -->|REST + JWT| API[Django REST Framework]
    API --> PLANNER[Planner tour_routes]
    PLANNER --> GEO[Nominatim geocoding]
    PLANNER --> OSRM[OSRM foot routing]
    PLANNER --> OVERPASS[Overpass POIs]
    PLANNER --> PLACES[(places.Place + PostGIS)]
    PLANNER --> CACHE[(RouteSearchCache)]
    API --> ENRICH[Wikidata / Wikipedia]
    ENRICH --> PLACES
```

---

## Stack

| Camada | Tecnologias |
|---|---|
| **Backend** | Python 3.12, Django 5.1, DRF 3.15, SimpleJWT 5.3, GeoDjango (`django.contrib.gis`), psycopg2 |
| **Banco** | PostgreSQL 16 + PostGIS 3.4 |
| **Frontend** | Expo SDK 52, React Native 0.76, React 18, TypeScript, React Navigation 7, React Native WebView, Reanimated |
| **Mapa** | Leaflet embarcado via HTML em `WebView` (mobile) / `iframe` (web) |
| **Infra local** | Docker + Docker Compose, `entrypoint.sh` no backend |
| **Fontes externas** | Nominatim (geocoding + extratags), OSRM (rota pedestre), Overpass (POIs), Wikidata (P18), Wikipedia (descrição/imagem) |
| **Docs / modelagem** | Mermaid, Modelio (UML), LaTeX (paper) |

---

## Como subir o ambiente completo

> Guia detalhado em [`docs/SETUP.md`](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/SETUP.md). Resumo abaixo.

### 1. Clonar os 3 repositórios lado a lado

Veja [Repositórios](#repositórios).

### 2. Configurar variáveis de ambiente

```bash
cp backend/.env.example  backend/.env
cp frontend/.env.example frontend/.env
```

Ajuste `DJANGO_SECRET_KEY` no `backend/.env`. O `frontend/.env` deve apontar para o backend (por padrão `EXPO_PUBLIC_API_URL=http://localhost:8080`).

### 3. Subir backend + banco

O `docker-compose.yml` mora no `backend/` e referencia o `frontend/` como caminho irmão.

```bash
cd backend
docker compose up --build -d
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py seed_demo --reset   # opcional
```

### 4. Verificar saúde da API

- Backend: <http://localhost:8080/api/health/>
- Admin Django: <http://localhost:8080/admin/>

### 5. Subir frontend

```bash
cd ../frontend
npm install
npm run web          # abre o Expo Web (geralmente em http://localhost:8081)
```

### Portas padrão

| Serviço | Host | Container |
|---|---|---|
| Backend (Django/Gunicorn) | `8080` | `8000` |
| PostgreSQL + PostGIS | `5433` | `5432` |
| Frontend (Expo Web via Docker) | `8082` | `8081` |

---

## Diagramas

Todos os diagramas estão versionados em [`paper-pce/figuras/`](https://github.com/EXPLORA-PLUS/explora-plus-docs/tree/main/paper-pce/figuras) e [`paper-pcg/figuras/`](https://github.com/EXPLORA-PLUS/explora-plus-docs/tree/main/paper-pcg/figuras) (exportações PNG para paper), em [`diagramas/mermaid/`](https://github.com/EXPLORA-PLUS/explora-plus-docs/tree/main/diagramas/mermaid) (fontes Mermaid editáveis) e no pacote final do Modelio [`explora-plus-diagramas-uml-modelio-final.zip`](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/explora-plus-diagramas-uml-modelio-final.zip) dentro do repo `explora-plus-docs`. Neste perfil, a preferência é mostrar os Mermaid diretamente para manter a documentação viva e fácil de atualizar.

### Casos de uso

```mermaid
flowchart LR
    Visitante((Visitante))
    Turista((Turista))

    Turista -.->|estende| Visitante

    Visitante --> UC1[Visualizar mapa com rota e POIs]
    Visitante --> UC2[Filtrar POIs por categoria\nCultura / Parques / Comida]
    Visitante --> UC3[Criar conta]
    Visitante --> UC4[Fazer login]

    Turista --> UC5[Gerar rota turistica\norigem e destino com POIs]
    Turista --> UC6[Abrir detalhe de POI\nimagem, resumo, horarios, website]
    Turista --> UC7[Marcar POI como visitado]
    Turista --> UC8[Desmarcar POI como visitado]
    Turista --> UC9[Excluir POI da rota atual]
    Turista --> UC10[Ver biblioteca pessoal de lugares]
    Turista --> UC11[Ver perfil basico]
    Turista --> UC12[Sair da conta]
    Turista --> UC13[Configurar busca\ncategorias, distancia e raio]
```

### Classes

```mermaid
classDiagram
    class User {
        +Long id
        +String username
        +String email
        +String passwordHash
        +DateTime dateJoined
        +authenticate(password) bool
    }

    class UserRouteSearchPreference {
        +Long id
        +bool includeCulture
        +bool includePark
        +bool includeFood
        +int poiSpacingM
        +int maxSearchRadiusM
        +appliesOnNextSearchOnly() bool
    }

    class PlaceCategory {
        +Long id
        +String slug
        +String name
        +String iconName
        +bool isActive
    }

    class Place {
        +Long id
        +String slug
        +String name
        +String sourceRef
        +String osmType
        +Long osmId
        +String wikidataId
        +String wikipediaTitle
        +Point location
        +String address
        +String summary
        +String openingHours
        +String website
        +String sourceUrl
        +String detailStatus
        +DateTime detailsFetchedAt
        +bool isActive
        +primaryImageUrl() String
        +distanceTo(point) float
    }

    class PlaceImage {
        +Long id
        +String url
        +int order
        +String caption
    }

    class UserPlaceState {
        +Long id
        +bool isVisited
        +DateTime visitedAt
        +DateTime firstSeenAt
        +DateTime lastSeenAt
        +int seenCount
        +markVisited() void
        +markUnvisited() void
    }

    class RouteSearchCache {
        +Long id
        +String cacheKey
        +dict searchPayload
        +String originQuery
        +String destinationQuery
        +dict routePayload
        +dict mapPayload
        +int hitCount
        +DateTime createdAt
        +bumpHit() void
    }

    class TourRoute {
        +Long id
        +String originLabel
        +String destinationLabel
        +Point originLocation
        +Point destinationLocation
        +String mode
        +int distanceM
        +int durationS
        +LineString routeGeometry
        +DateTime createdAt
    }

    class TourRouteStop {
        +Long id
        +int displayOrder
        +int waypointOrder
        +String state
        +float distanceFromRouteM
    }

    User "1" --> "1" UserRouteSearchPreference : configura
    User "1" --> "0..*" TourRoute : cria
    User "1" --> "0..*" UserPlaceState : possui
    PlaceCategory "1" --> "0..*" Place : categoriza
    Place "1" *-- "0..*" PlaceImage : tem
    Place "1" --> "0..*" UserPlaceState : rastreada_em
    Place "1" --> "0..*" TourRouteStop : aparece_em
    RouteSearchCache "1" --> "0..*" TourRoute : origina
    TourRoute "1" *-- "0..*" TourRouteStop : contem
    TourRoute "0..1" --> "0..*" UserPlaceState : ultima_rota_vista
```

### Modelo Entidade-Relacionamento

```mermaid
erDiagram
    USER ||--|| USER_ROUTE_SEARCH_PREFERENCE : configura
    USER ||--o{ TOUR_ROUTE : cria
    USER ||--o{ USER_PLACE_STATE : possui

    PLACE_CATEGORY ||--o{ PLACE : categoriza

    PLACE ||--o{ PLACE_IMAGE : tem
    PLACE ||--o{ USER_PLACE_STATE : rastreada_em
    PLACE ||--o{ TOUR_ROUTE_STOP : aparece_em

    ROUTE_SEARCH_CACHE ||--o{ TOUR_ROUTE : origina

    TOUR_ROUTE ||--o{ TOUR_ROUTE_STOP : contem
    TOUR_ROUTE }o--o{ USER_PLACE_STATE : ultima_rota_vista

    USER {
        bigint id PK
        string username UK
        string email UK
        string password_hash
        datetime date_joined
    }

    USER_ROUTE_SEARCH_PREFERENCE {
        bigint id PK
        bigint user_id FK
        bool include_culture
        bool include_park
        bool include_food
        smallint poi_spacing_m "75 | 100 | 150"
        smallint max_search_radius_m "150 | 250 | 400"
        datetime created_at
        datetime updated_at
    }

    PLACE_CATEGORY {
        bigint id PK
        string slug UK "culture | park | food"
        string name
        string icon_name
        bool is_active
    }

    PLACE {
        bigint id PK
        string slug UK
        bigint category_id FK
        string name
        string source_ref UK "stop_id publico"
        string osm_type "node | way | relation"
        bigint osm_id
        string wikidata_id
        string wikipedia_title
        point location "PostGIS Point (lng,lat)"
        string address
        text summary
        string opening_hours
        string website
        string source_url
        string detail_status "pending|complete|unavailable|error"
        datetime details_fetched_at
        bool is_active
        datetime created_at
        datetime updated_at
    }

    PLACE_IMAGE {
        bigint id PK
        bigint place_id FK
        string url
        int order
        string caption
    }

    USER_PLACE_STATE {
        bigint id PK
        bigint user_id FK
        bigint place_id FK
        bool is_visited
        datetime visited_at
        datetime first_seen_at
        datetime last_seen_at
        int seen_count
        bigint last_seen_route_id FK "nullable"
    }

    ROUTE_SEARCH_CACHE {
        bigint id PK
        string cache_key UK
        json search_payload
        string origin_query
        string destination_query
        json route_payload
        json map_payload
        int hit_count
        datetime created_at
        datetime updated_at
    }

    TOUR_ROUTE {
        bigint id PK
        bigint user_id FK
        bigint search_cache_id FK
        string origin_label
        string destination_label
        point origin_location
        point destination_location
        string mode "tour | direct_fallback"
        int distance_m
        int duration_s
        int direct_distance_m
        int direct_duration_s
        linestring route_geometry
        linestring direct_route_geometry
        datetime created_at
        datetime updated_at
    }

    TOUR_ROUTE_STOP {
        bigint id PK
        bigint route_id FK
        bigint place_id FK
        int display_order
        int waypoint_order "null se nao esta na rota"
        string state "active | visited | excluded"
        string source
        float distance_from_route_m
        datetime created_at
        datetime updated_at
    }
```

### Atividade — fluxo Explorar

```mermaid
flowchart TD
    Start([Inicio]) --> OpenApp[Abrir app]
    OpenApp --> AuthCheck{Sessao\nautenticada?}

    AuthCheck -->|Sim| TryCurrentRoute[GET /api/tour-routes/current/]
    TryCurrentRoute --> RouteFound{Rota salva\nencontrada?}
    RouteFound -->|200 OK| ShowRoute[Exibe rota salva]
    RouteFound -->|404| GenDefault[Gera rota padrao\nPOST /api/tour-routes/]

    AuthCheck -->|Nao| GenAnon[Gera rota anonima\nPOST /api/tour-routes/]

    ShowRoute --> RenderMap
    GenDefault --> RenderMap
    GenAnon --> RenderMap[Renderiza mapa Leaflet\npolyline + marcadores]

    RenderMap --> Prefetch[Pre-busca detalhes de todos os POIs\nem background com 400ms de intervalo]
    Prefetch --> Idle{Aguarda\ninteracao}

    Idle -->|Filtro categoria| Filter[Atualiza visibilidade\ndos marcadores]
    Filter --> Idle

    Idle -->|Toca marcador ou item| OpenDetail[GET pois/stop_id\ninstantaneo se pre-buscado]
    OpenDetail --> Modal[Exibe TourPoiDetailModal]

    Modal -->|Marcar visitado| PatchVisited[PATCH .../state/ visited]
    PatchVisited --> UpdateRoute[Atualiza rota e mapa]
    UpdateRoute --> CloseModal

    Modal -->|Excluir da rota| DeleteStop[DELETE .../stops/stop_id/]
    DeleteStop --> RebuildRoute[Reconstroi rota]
    RebuildRoute --> CloseModal

    Modal -->|Fechar| CloseModal[Fecha modal]
    CloseModal --> Idle

    Idle -->|Abrir perfil| Profile[ProfileScreen]
    Profile -->|Abrir configuracoes| Settings[GET /api/tour-routes/preferences/]
    Settings --> Adjust[Alterna categorias\nescolhe distancia e raio]
    Adjust --> SaveSettings[PATCH /api/tour-routes/preferences/]
    SaveSettings --> Idle

    Idle -->|Recalcular rota| NewRoute[POST /api/tour-routes/\nusa preferencias salvas]
    NewRoute --> RenderMap
```

### Sequência — gerar rota

```mermaid
sequenceDiagram
    actor T as Turista
    participant App as ExploreScreen
    participant API as Backend API
    participant DB as PostgreSQL
    participant Cache as RouteSearchCache
    participant Nom as Nominatim
    participant OSRM as OSRM
    participant OVP as Overpass API

    T->>App: Preenche origem e destino, toca "Gerar Rota"
    App->>API: POST /api/tour-routes/ {origin, destination}

    alt Usuario autenticado
        API->>DB: Carrega UserRouteSearchPreference
        DB-->>API: categorias + poi_spacing_m + max_search_radius_m
    else Usuario anonimo
        API->>API: Usa defaults do planner
    end

    API->>Cache: Busca cache_key com origem, destino e preferencias efetivas
    alt Cache valido encontrado
        Cache-->>API: route_payload + map_payload
        API->>DB: Upsert Place para cada POI do cache
    else Cache miss ou fallback ruim
        API->>Nom: Geocodifica origem e destino
        Nom-->>API: {lat, lng} para cada ponto
        API->>OSRM: GET /route/v1/foot/{coords}
        OSRM-->>API: polyline + distance_m + duration_s
        API->>OVP: Busca POIs com categorias e raio filtrados
        OVP-->>API: Lista de nos OSM (nome, coord, tags)
        API->>API: Seleciona POIs com spacing configurado
        API->>DB: Upsert Place para cada POI encontrado
        API->>Cache: INSERT RouteSearchCache com search_payload, route_payload e map_payload
    end

    alt Usuario autenticado
        API->>DB: INSERT TourRoute (user, cache, geometria)
        API->>DB: INSERT TourRouteStop para cada POI (state=active|visited)
        API->>DB: INSERT/UPDATE UserPlaceState para cada POI
        API-->>App: 200 {route, map, saved_route_id}
    else Usuario anonimo
        API-->>App: 200 {route, map}
    end

    App->>App: Renderiza polyline e marcadores no mapa Leaflet
    App-->>T: Exibe rota com paradas, distancia e duracao
```

### Sequência — detalhe de POI

```mermaid
sequenceDiagram
    actor T as Turista
    participant App as ExploreScreen
    participant API as Backend API
    participant DB as PostgreSQL
    participant Nom as Nominatim
    participant WD as Wikidata API
    participant WP as Wikipedia API

    T->>App: Toca em marcador do mapa ou item da lista
    App->>API: GET /api/tour-routes/pois/{stop_id}/
    API->>DB: SELECT Place WHERE source_ref = stop_id

    alt Detalhes incompletos (details_fetched_at IS NULL ou todos campos vazios)
        API->>Nom: Lookup por osm_type/osm_id com extratags=1
        Nom-->>API: display_name, website, opening_hours, wikidata, wikipedia

        alt wikidata_id disponivel
            API->>WD: GET /w/api.php?action=wbgetentities&ids=Qxxx
            WD-->>API: claims P18 (imagem), sitelinks (titulo Wikipedia)
        end

        alt wikipedia_title ainda nao encontrado
            API->>WP: GET /w/api.php?action=query&list=search&srsearch={nome}
            WP-->>API: Primeiro resultado da busca por nome (pt, depois en)
        end

        alt wikipedia_title disponivel
            API->>WP: GET /api/rest_v1/page/summary/{title}
            WP-->>API: extract (resumo), thumbnail/originalimage, content_urls
        end

        API->>DB: UPDATE Place (address, summary, website, opening_hours, source_url, wikipedia_title, detail_status)
        API->>DB: INSERT PlaceImage (url da imagem encontrada)
    end

    API-->>App: {stop_id, name, category, address, summary, image_url, opening_hours, website, source_url}
    App-->>T: Exibe modal TourPoiDetailModal com detalhes enriquecidos
```

### Sequência — marcar como visitado

```mermaid
sequenceDiagram
    actor T as Turista
    participant App as ExploreScreen
    participant API as Backend API
    participant DB as PostgreSQL
    participant Planner as Route Planner

    T->>App: Toca "Marcar como visitado" no modal do POI
    App->>API: PATCH /api/tour-routes/saved/{route_id}/stops/{stop_id}/state/ {state: "visited"}

    API->>DB: SELECT TourRoute WHERE id=route_id AND user=request.user
    API->>DB: Verifica se stop_id esta na base_route_payload

    API->>DB: UPDATE UserPlaceState SET is_visited=True, visited_at=now()
    API->>DB: UPDATE TourRouteStop SET state="visited"

    API->>Planner: Reconstroi rota excluindo stops visitados do trajeto ativo
    Planner->>DB: SELECT stops WHERE state != "visited" AND state != "excluded"
    Planner-->>API: Nova polyline + waypoints recalculados

    API->>DB: UPDATE TourRoute (route_geometry, distance_m, duration_s)
    API-->>App: 200 {route atualizada com novo trajeto}

    App->>App: Move POI para secao "Ja visitados"
    App->>App: Atualiza marcador no mapa para estado "visited"
    App-->>T: Exibe rota atualizada sem o POI no trajeto ativo
```

---

## Colaboradores

Equipe extraída do histórico Git dos três repositórios (`git log --all`).

| Nome | Papel | E-mail | Repositórios |
|---|---|---|---|
| **Lucas Cerqueira Galvão** | Desenvolvimento (backend + frontend + docs) | <lucasgalvao134@gmail.com> | backend · frontend · docs |
| **Lucas Carmona Neto** | Desenvolvimento (backend + frontend + docs) | <lucascarmonaneto510@gmail.com> | backend · frontend · docs |
| **João Gabriel Catalão** | Desenvolvimento (backend) | <joaogabriel3556@gmail.com> | backend |
| **Felipe Barbosa dos Santos** | Documentação e modelagem | <felipebsantos@unisantos.br> | docs |
| **Felipe Monteiro** | Documentação e modelagem | <felipemonteiro@unisantos.br> | docs |

---

## Status

| Fase | Estado |
|---|---|
| Arquitetura central (roteamento + enriquecimento de POIs + personalização) | ✅ Validada |
| Aprovação do PO e demonstração do protótipo | ✅ Concluída |
| Modelo de monetização e estratégia de distribuição | 🔜 Próxima fase |
| Integrações com plataformas de eventos e mobilidade (Sympla, Uber etc.) | 🔜 Pós-monetização |

---

## Instituição

Projeto desenvolvido na **Universidade Católica de Santos (UniSantos)** como Projeto de Conclusão (PCE).  
Organização GitHub: <https://github.com/EXPLORA-PLUS>  
Paper completo em [`paper/main.pdf`](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/paper/main.pdf) · proposta original em [`paper/proposta-original.pdf`](https://github.com/EXPLORA-PLUS/explora-plus-docs/blob/main/paper/proposta-original.pdf).
