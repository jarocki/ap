# Adversary Pursuit: Design and Architecture of a Gamified Cyber Threat Intelligence and OSINT CLI Framework

*   **Key Points**
    *   Research suggests that combining the interactive, module-driven architecture of Metasploit with the gamified, objective-based progression of CTFd could significantly enhance the engagement and skill acquisition of cybersecurity practitioners.
    *   It seems likely that the most robust foundation for a Python 3.12+ interactive command-line interface (CLI) is the `cmd2` framework integrated with `rich` for terminal formatting and `prompt_toolkit` for readline history, as opposed to pure Text User Interface (TUI) libraries like `textual`.
    *   The evidence leans toward using `importlib.metadata` and Python Entry Points (`entry_points.txt` or `pyproject.toml`) combined with `typing.Protocol` to establish a scalable, side-effect-free plugin architecture.
    *   Existing Threat Intelligence (TI) platforms, notably OpenCTI and IntelOwl, demonstrate that an intermediate graph-based data model (such as STIX 2.1) and a microservice-inspired modular design are essential for complex Open Source Intelligence (OSINT) pivoting.
    *   Implementing a dynamic scoring system with parabolic decay models, mirroring CTFd, is generally considered an effective method to balance challenge difficulty and extrinsic motivation organically.

### Executive Summary
The domain of Cyber Threat Intelligence (CTI) and Open Source Intelligence (OSINT) gathering often suffers from a steep learning curve and disjointed tooling. Analysts are frequently required to navigate a myriad of disconnected scripts, web portals, and Application Programming Interfaces (APIs). The proposed framework, "Adversary Pursuit," seeks to bridge this gap by synthesizing the familiar, tactile command-line experience of the Metasploit Framework with the engaging, pedagogical mechanics of Capture The Flag (CTF) platforms. This report explores the architectural design required to instantiate Adversary Pursuit as a version 1 (v1) multi-platform Python 3.12+ CLI application.

### Methodological Approach
This architectural blueprint is derived from an exhaustive analysis of existing open-source ecosystems. The design incorporates research into Python-centric CLI frameworks, gamification theories applied to cybersecurity education, data standardization protocols (STIX 2.1), and advanced plugin discovery mechanisms. By dissecting the structural paradigms of established tools such as SpiderFoot, TheHive, IntelOwl, and Maltego, this document formulates a unified software engineering strategy tailored for modern Python development.

***

## 1. Interface and CLI Framework Evaluation for Python 3.12+

The foundational layer of the Adversary Pursuit framework requires an interactive console that mimics the immersive, stateful environment of Metasploit (`msfconsole`). The interface must support workspaces, session management, dynamic module loading, tab completion, and visually appealing output formatting [cite: 1, 2].

### 1.1 Comparative Analysis of Python CLI Libraries
When evaluating Python libraries for building interactive, line-oriented command processors, developers must choose between traditional command loop processors and modern Text User Interfaces (TUIs).

*   **cmd and cmd2**: The standard Python library provides the `cmd` module, a bare-bones framework for line-oriented command interpreters [cite: 3, 4]. However, `cmd2` acts as a "batteries-included" extension, transitioning developers from the "bronze Age to Iron Age in terms of interactive cli" [cite: 4]. It is highly recommended for applications operated primarily via interactive CLI [cite: 4]. `cmd2` provides robust history management, alias creation, macro execution, and powerful tab completion natively [cite: 4, 5]. Crucially, it utilizes `prompt_toolkit` to provide a pure Python, cross-platform readline experience [cite: 3].
*   **Textual**: While `Textual` is a highly capable framework for building complex TUIs, community consensus indicates that it currently lacks robust support for standard terminal use cases like a native REPL (Read-Eval-Print Loop) console [cite: 6]. Attempting to embed a `cmd2` instance inside a Textual widget is notoriously difficult and often fails to replicate the true standard input/output terminal feel expected from a Metasploit clone [cite: 6].
*   **Rich**: `Rich` is designed to generate beautiful, formatted terminal output, including tables, syntax highlighting, and progress bars [cite: 7].

### 1.2 The Selected Stack: cmd2 + Rich
To achieve the Metasploit feel, Adversary Pursuit will utilize `cmd2` as the core execution loop and `rich` as the rendering engine. `cmd2` natively supports Rich Consoles through specialized base classes such as `Cmd2BaseConsole`, `Cmd2GeneralConsole`, and `Cmd2ExceptionConsole` [cite: 8]. This allows the framework to handle core routing logic based on global settings (e.g., `ALLOW_STYLE` and `APP_THEME`) while preserving Rich's layout capabilities [cite: 8]. 

By inheriting from `cmd2.Cmd`, Adversary Pursuit can immediately instantiate a command prompt (e.g., `adversary-pursuit> `) and provide built-in commands, scriptability, and pipeline redirection [cite: 4, 7]. This satisfies the requirement for a Metasploit-like `msfconsole` where users can type `use osint/domain/subdomain_enum`, altering the state of the prompt and loading specific module contexts [cite: 9].

## 2. Metasploit-Inspired Architectural Paradigms

Metasploit's success as a penetration testing framework stems from its rigid yet highly flexible internal architecture. Adversary Pursuit will adopt several of these conceptual models to organize CTI and OSINT workflows [cite: 1, 2].

### 2.1 Modular Namespace and Command Structure
Metasploit categorizes its functionalities into distinct repositories: exploits, payloads, and auxiliary modules [cite: 1]. Adversary Pursuit will utilize a similar namespace hierarchy:
*   `modules/osint/`: Modules that query open-source public APIs without alerting the target (e.g., WHOIS, Shodan).
*   `modules/cti/`: Modules that query threat intelligence platforms (e.g., VirusTotal, OpenCTI, MISP) for known indicators.
*   `modules/pivoting/`: Logic that takes an artifact (e.g., an IP) and maps it to related artifacts (e.g., domains, SSL certificates) mirroring Maltego's transforms [cite: 10, 11].

Commands will replicate the Metasploit syntax:
*   `search [keyword]`: Discovers available OSINT/CTI modules [cite: 9].
*   `use [module_path]`: Changes the context of the `cmd2` console to the specific module [cite: 9].
*   `show options`: Displays required variables (e.g., `API_KEY`, `TARGET_IP`) [cite: 9].
*   `set [variable] [value]`: Configures the module parameters [cite: 9].
*   `run` or `exploit` (alias `hunt`): Executes the module logic [cite: 9].

### 2.2 Workspaces and Datastore
Metasploit utilizes a PostgreSQL backend (`msfdb`) to separate engagement data into workspaces, ensuring that targets, credentials, and notes from one pentest do not bleed into another [cite: 1, 12]. Adversary Pursuit will implement a SQLite or PostgreSQL database via an Object-Relational Mapper (ORM) to create **Workspaces**. Each workspace will act as an isolated graph, storing STIX 2.1 formatted cyber observables (SCOs) discovered during a specific hunting session. 

### 2.3 Sessions
In Metasploit, a "session" typically refers to an active command shell or Meterpreter connection established on a compromised host [cite: 12]. In the context of Adversary Pursuit, a **Session** will represent an active, authenticated connection to a third-party intelligence stream (e.g., an active WebSocket to a MISP instance or a long-running SpiderFoot correlation task). Users will be able to background these intelligence-gathering tasks and interact with them using a `sessions -i [id]` command.

## 3. Extensible Plugin and Module System Design (Python 3.12+)

To ensure Adversary Pursuit is extensible, it must implement a robust plugin architecture. The historical approach of using `importlib.import_module` alongside `pkgutil` to scan filesystem directories for plugins is considered an anti-pattern in modern software engineering [cite: 13]. Scanning directories forces implicit import ordering, risks import-time side effects that can crash the core application, and tightly couples the core system to internal plugin implementations [cite: 13].

### 3.1 Entry Points and importlib.metadata
A scalable plugin system treats plugins as versioned packages, explicitly enforcing boundaries [cite: 13]. Adversary Pursuit will leverage Python's `importlib.metadata` library and the Entry Points interoperability specification [cite: 14, 15]. Entry points allow an installed distribution to advertise the components it provides so that they can be discovered dynamically at runtime [cite: 15].

In the `pyproject.toml` or `entry_points.txt` of a third-party plugin, the developer will declare:
```toml
[project.entry-points."adversary_pursuit.modules"]
shodan_ip = "ap_shodan_plugin.main:ShodanIPLookup"
virustotal_hash = "ap_vt_plugin.main:VTHashCheck"
```

The core Adversary Pursuit framework will load these modules safely without blindly executing file system paths [cite: 13, 16]:
```python
from importlib.metadata import entry_points
from typing import Dict, Type

def discover_modules() -> Dict[str, Type]:
    discovered_modules = {}
    # Python 3.10+ select interface
    eps = entry_points(group="adversary_pursuit.modules") 
    for ep in eps:
        try:
            # load() resolves the value to the actual class/function
            plugin_class = ep.load() 
            discovered_modules[ep.name] = plugin_class
        except Exception as e:
            print(f"Failed to load plugin {ep.name}: {e}")
    return discovered_modules
```
This architecture ensures that plugins are versioned, installation is explicit via package managers (like `pip`), and import errors are strictly isolated to the failing plugin rather than taking down the entire `cmd2` application [cite: 13, 15].

### 3.2 Defining the Plugin Protocol
To maintain strict contracts between the core framework and third-party modules, Adversary Pursuit will utilize `typing.Protocol` (structural subtyping) instead of massive Abstract Base Classes [cite: 13]. Using `Protocol` ensures plugins remain lightweight while strictly adhering to the expected input and output signatures [cite: 13].

```python
from typing import Protocol, Any, Dict

class PursuitModule(Protocol):
    name: str
    description: str
    author: str
    
    def initialize(self, config: Dict[str, Any]) -> None:
        """Called to initialize with API keys/config without triggering execution."""
        ...
        
    def hunt(self, target: str, options: Dict[str, Any]) -> Dict[str, Any]:
        """The core execution logic returning STIX 2.1 compliant observables."""
        ...
```
This approach guarantees that plugin initialization remains side-effect free, separating the loading mechanism from the execution mechanism—a critical requirement for predictable startup times and safe failure handling [cite: 13].

## 4. CTFd-Inspired Gamification and Progression Engine

A fundamental differentiator of Adversary Pursuit is its integration of gamification to foster intrinsic and extrinsic motivation [cite: 17]. Borrowing from the architecture of CTFd—a ubiquitous open-source CTF platform—the framework will incorporate scoring, challenges, progression logic, and a virtual economy [cite: 17, 18].

### 4.1 Challenge Types and Design
CTFd supports various challenge structures beyond traditional string submission, including multiple choice, manual verification, and coding tasks [cite: 19]. In Adversary Pursuit, a "Challenge" is contextualized as an intelligence requirement. 
*   **Standard Challenges**: Finding a specific IP address associated with a known APT group's domain.
*   **Pivoting Challenges**: Starting with a single email address and successfully extracting a target's physical location through Maltego-style multi-step transforms [cite: 10, 11].

### 4.2 Dynamic Scoring Architecture
To accurately reflect the difficulty of an intelligence gathering task, Adversary Pursuit will implement Dynamic Scoring. In dynamically scored challenges, the point value starts at a high initial tier and decreases based on the number of users (or teams) that successfully solve it [cite: 20, 21]. This naturally calibrates the point valuation, ensuring that heavily solved (easier) challenges reward fewer points [cite: 20, 22].

Adversary Pursuit will implement CTFd's parabolic decay function [cite: 20]. A parabolic function (as opposed to linear or logarithmic) ensures that higher-valued challenges experience a slower drop-off from their initial value, maintaining high stakes for early solvers [cite: 20, 22].

The mathematical logic is modeled as follows:
\[ \text{value} = \left( \frac{\text{minimum} - \text{initial}}{\text{decay}^2} \right) \times (\text{solve\_count}^2) + \text{initial} \]

```python
import math

def calculate_dynamic_score(initial: int, minimum: int, decay: int, solve_count: int) -> int:
    """
    Calculates the dynamic score based on CTFd parabolic decay logic.
    """
    if solve_count >= decay:
        return minimum
        
    value = (((minimum - initial) / (decay ** 2)) * (solve_count ** 2)) + initial
    value = math.ceil(value)
    
    return max(value, minimum)
```
The system tracks three core variables per challenge:
1.  **Initial**: Original point valuation [cite: 20, 22].
2.  **Decay**: The threshold number of solves required before the challenge hits its minimum value [cite: 20, 22].
3.  **Minimum**: The absolute lowest point value the challenge can reach [cite: 20, 22].

### 4.3 Hint System and Virtual Economy
To prevent practitioners from becoming permanently stuck, the framework will include a Hint system. Following CTFd's architecture, hints can be either free or carry a point cost [cite: 23]. If a user unlocks a paid hint, their aggregate score drops by the hint's cost [cite: 23]. Users cannot unlock hints if their score is insufficient, preventing negative point balances [cite: 23]. 

### 4.4 Leaderboards and Badges
Adversary Pursuit will feature a local or networked database storing team profiles, recording First Bloods (first to solve), and calculating Elo ratings or simple point aggregations for a live leaderboard. Badges (achievements) will be awarded for behavioral milestones, such as executing a successful 5-step OSINT pivot without relying on a hint [cite: 17, 18].

## 5. Integrating Existing Open-Source CTI and OSINT Architectures

To build a comprehensive intelligence tool, Adversary Pursuit must stand on the shoulders of giants. The framework's internal architecture will draw heavily from the operational patterns of established tools like OpenCTI, TheHive, IntelOwl, SpiderFoot, and Maltego.

### 5.1 Data Modeling: The OpenCTI and STIX 2.1 Standard
OpenCTI is a centralized platform designed to manage CTI data using a knowledge schema built on STIX 2.1 standards [cite: 24, 25]. The most critical architectural lesson from OpenCTI is that it operates as a transformation engine, not merely a passive database [cite: 26, 27].

When Adversary Pursuit ingests data from a module, it must convert raw JSON into STIX Domain Objects (SDO), STIX Cyber Observables (SCO), and STIX Relationship Objects (SRO) [cite: 28]. 
*   **SDOs** represent high-level intelligence concepts: Threat Actors, Attack Patterns, Malware [cite: 28].
*   **SCOs** represent technical evidence: IP Addresses, Domain Names, File Hashes [cite: 28].
*   **SROs** define the links: e.g., an SRO connects a Malware SDO to an IP Address SCO (indicating command and control infrastructure) [cite: 28].

By adopting STIX 2.1 internally, Adversary Pursuit ensures that data collected from a Shodan query can be seamlessly mapped, exported, and correlated against data collected from VirusTotal.

### 5.2 Case Management and Observables: TheHive
TheHive operates as a Security Incident Response Platform (SIRP) [cite: 29]. Its architecture relies heavily on turning alerts into actionable cases and extracting "observables" for enrichment [cite: 29, 30]. Adversary Pursuit's workspace feature will mimic TheHive's Case Management. When an analyst starts a hunt, they open a "Case." All OSINT indicators discovered during this session are tagged as "Observables" and attached to the Case timeline, providing an immutable audit trail of the investigation [cite: 29, 31].

### 5.3 Microservices and Analyzers: IntelOwl
IntelOwl automates the collection of threat intelligence by querying multiple sources simultaneously via a single API [cite: 32, 33]. IntelOwl's architecture is strictly modular, dividing tasks into:
*   **Analyzers**: Retrieve data from external sources (VirusTotal) or internal tools (Yara) [cite: 33, 34].
*   **Connectors**: Export data to external platforms like MISP [cite: 33, 34].
*   **Playbooks**: Define a highly abstracted flow (e.g., given an IP -> run AbuseIPDB analyzer -> run Shodan analyzer -> push to MISP connector) [cite: 33, 35].

Adversary Pursuit will implement an internal "Playbook" scripting language, allowing users to write simple text files that chain modules together sequentially, mimicking IntelOwl's automated enrichment flows [cite: 35, 36].

### 5.4 OSINT Automation and Publisher/Subscriber Models: SpiderFoot
SpiderFoot automates OSINT collection using over 200 modules to map attack surfaces [cite: 37, 38]. Its architecture relies on a core correlation engine that uses a publisher/subscriber model [cite: 38]. When one module discovers an email address, it publishes that artifact to an internal event bus. Other modules subscribed to "email addresses" (e.g., HaveIBeenPwned modules) automatically trigger, creating a cascading web of intelligence gathering [cite: 38].

Adversary Pursuit will feature an `auto_pivot` setting within workspaces. If enabled, the core loop will utilize an internal event bus (potentially implemented via standard Python `asyncio` queues) to automatically feed discovered SCOs into relevant, enabled OSINT modules.

### 5.5 Visual Link Analysis and Pivoting: Maltego
Maltego excels at visual link analysis using "Transforms" (TDS or Local) [cite: 10, 11]. A transform is a script that takes an entity, queries a data source, and returns related entities, expanding a graph visually [cite: 10]. Although Adversary Pursuit is a CLI tool, it will implement a "Graph State" memory. Users can execute `pivot --target [Entity_ID] --transform [Module_Name]`. The resulting data will be stored as SROs (relationships), allowing the CLI to render text-based relationship trees or export the graph via Graphviz/GEXF for external viewing [cite: 11, 38].

## 6. Comprehensive OSINT and CTI API Data Sources

An OSINT framework is only as powerful as the data it can access. Adversary Pursuit will include native support (via the plugin architecture) for APIs curated in repositories like `awesome-threat-intelligence` and `awesome-osint` [cite: 39, 40]. 

### 6.1 Vulnerability and Attack Surface Metrics
*   **Shodan / Censys**: Search engines for Internet-connected devices, returning banners, open ports, and associated CVEs [cite: 40].
*   **VulnCheck / NVD**: For assessing vulnerabilities and tracking exploit availability [cite: 41].

### 6.2 IP, Domain, and DNS Intelligence
*   **VirusTotal**: Comprehensive file, URL, and IP reputation database aggregating multiple antivirus engines [cite: 34, 39].
*   **AbuseIPDB**: Community-driven blacklist for reporting IP addresses associated with malicious activity [cite: 39].
*   **PassiveTotal / RiskIQ**: Advanced passive DNS, WHOIS history, and infrastructure profiling, crucial for tracking threat actor movements across hosting providers.
*   **Spamhaus**: Authoritative IP and domain reputation source (SBL, DBL) [cite: 41].

### 6.3 Data Breach and Social Media OSINT
*   **HaveIBeenPwned (HIBP) / CredenShow / StealSeek**: For checking if specific email addresses or domains appear in known data dumps [cite: 40].
*   **Sherlock**: A widely used OSINT tool to search for user accounts across hundreds of social networks, highly effective for social engineering footprinting [cite: 10].

### 6.4 Threat Actor and IOC Aggregators
*   **AlienVault OTX**: Open access to a global community of threat researchers delivering community-generated threat data [cite: 39, 42].
*   **Ransomwatch**: Open-source ransomware tracking monitoring active extortion group leak sites [cite: 41].

## 7. Practical Implementation Plan for Adversary Pursuit v1

Integrating the discussed paradigms into a cohesive Python 3.12+ application requires strict structural organization.

### 7.1 Directory and Package Structure
The application will be packaged as a standard Python module using modern `pyproject.toml` standards.

```text
adversary_pursuit/
├── pyproject.toml              # Build system, dependencies (cmd2, rich, stix2)
├── src/
│   └── adversary_pursuit/
│       ├── __init__.py
│       ├── core/
│       │   ├── console.py      # The cmd2 entry point and Rich integration
│       │   ├── workspace.py    # Database connection, Graph/STIX state
│       │   └── plugin_mgr.py   # importlib.metadata entry point loading
│       ├── gamification/
│       │   ├── scoring.py      # Dynamic scoring math, hints, decay logic
│       │   └── challenges.py   # Challenge definitions and flag checking
│       ├── models/
│       │   └── stix.py         # STIX 2.1 abstraction layer
│       └── base_plugins/       # Core modules shipped natively
│           ├── osint/
│           ├── cti/
│           └── pivoting/
└── tests/
```

### 7.2 The Console Core (cmd2 + Rich)
The core CLI will initialize `cmd2` and override standard output to utilize `rich` tables and formatted text [cite: 4, 7].

```python
import cmd2
from rich.console import Console
from rich.table import Table
from adversary_pursuit.core.plugin_mgr import PluginManager

class APConsole(cmd2.Cmd):
    """The main Adversary Pursuit interactive console."""
    
    def __init__(self):
        super().__init__()
        self.prompt = '[bold red]ap>[/bold red] '
        self.rich_console = Console()
        self.plugin_manager = PluginManager()
        self.plugin_manager.load_plugins()
        self.current_workspace = "default"
        self.active_module = None
        
        self.intro = """
        [bold magenta]Adversary Pursuit Framework v1.0[/bold magenta]
        Type 'help' or '?' to list commands.
        """

    def do_use(self, args):
        """Load a specific OSINT/CTI module: use osint/shodan"""
        module_name = args.strip()
        if module_name in self.plugin_manager.plugins:
            self.active_module = self.plugin_manager.plugins[module_name]()
            self.prompt = f'ap [blue]{module_name}[/blue]> '
            self.rich_console.print(f"[green]Loaded module {module_name}[/green]")
        else:
            self.rich_console.print("[red]Module not found.[/red]")

    def do_run(self, args):
        """Execute the currently loaded module."""
        if not self.active_module:
            self.rich_console.print("[red]No module loaded.[/red]")
            return
            
        # Execute plugin logic and return STIX 2.1 observable
        results = self.active_module.hunt()
        
        # Gamification hook
        self._check_challenges(results)
```

### 7.3 The Gamification Event Hook
Whenever an OSINT or CTI module returns an artifact (e.g., an IP address, a hashed password), the application intercepts this data before storing it in the workspace datastore. It passes the artifact to the Gamification Engine to check against active challenges.

```python
def _check_challenges(self, observable: str):
    """Check if discovered observable matches any active challenge flags."""
    for challenge in self.active_challenges:
        if challenge.verify_flag(observable):
            pts = challenge.award_points()
            self.rich_console.print(f"[bold yellow]Challenge Solved! You earned {pts} points.[/bold yellow]")
```

## Conclusion
The architectural design of the "Adversary Pursuit" framework successfully maps the robust, tactical command-line usability of the Metasploit Framework onto the engaging, pedagogical structure of CTFd. By leveraging modern Python 3.12+ features—specifically `cmd2` for interactive loop processing, `rich` for interface rendering, and `importlib.metadata` for secure plugin extensibility—the application guarantees high performance and deep customizability. 

Furthermore, by adopting the STIX 2.1 data schema utilized by leading intelligence platforms like OpenCTI, and implementing the microservice analyzer logic of IntelOwl alongside the publisher/subscriber OSINT mechanics of SpiderFoot, Adversary Pursuit is uniquely positioned as a premier, gamified intelligence-gathering ecosystem. Through dynamic parabolic scoring and hint economies, it promises to effectively train the next generation of security analysts while providing a functional, production-ready OSINT toolkit.

**Sources:**
1. [securedebug.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHb5Ancr92ZS8eD858Gw2I2AnXML6j18L_zgXcDzLi0IoUo-ARYPj3aDD0jSjkSEQ86BWrIjOVjBxcaXXjER3qSDN-seusn4DgOmRLMxsa8xwC9gt5r3y5nVtYpgJ2_WqP8I9wPYjAlcrngxm3ArQK_6IAnMSmfidhG_UwjWKk=)
2. [imperva.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE6cJ5YvRLAPvKLzKtjyjnejneWv3wz_bz_KxYqbd0H0XSspB2hinGx1JgP8m_ZUriHsm-wx9oSzsycPOD6lbjeXcfjf9bnlIENppkpkUBbVJB5wqwCOhtck66f94bHEh7bKIF7U7Nyulp4AQATU6ygdfRGpw==)
3. [readthedocs.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQG6T0i6PEw3BmQnc16rIyASi5tlZUmtNMW9KzBOBpIw7NsDF-Xm2W5YVEvjHsrN1IDfQnIbrh6DA2ou0CLI29jhfyWPZ3ntj-SpoO_LGgTxj1-pzH9fupy4QRu0GU0NoS12qHa-TB0EhSwQ)
4. [plainenglish.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGkZgFVv_niaSbwkG5jBLeOlM3g_6Fm9_fwIZdID3XkJvtsJOXWG6johVnww85LR_VV4_03FXHYqecUF4wiTYOhYuQYDrXn1Dr-IdApzjOHELZBzLEbkB6FsUIYEJPiQFD4htzvIENa1JKg-ULiNBIbEX07)
5. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQG4_Yez_Rq1F6qvenTu5gVOxTODd9Z3OhcO-4jEITZ39PVDXV8g9urECu_OswZcm_RQWo4XNzsOnYtltlyuV40KcyvTpAm4GhGNElQgFTJLp4BlFqtiqa-4PA==)
6. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFesz9KI98fi7k8vMJjJBWxugua_V_x14CjCyITs65aG4QSzgFc3HndCpvEbBGMci8C1svQAtgP6Lfjhgx_zs7O66fD2V8IUT6T8HG3GkALN89lxM_INEpYtjh8-sDQwHpjrWxaJYu5474NbjY=)
7. [mteke.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGoL5RLJAdXksEvTGeT76EbV2kJKYo1qCh2gv1v43jtzAECKmIg479EQXG-Rdfit9WD8DtpFu6JBN1i26neeXg-jrew2x8F_rGEDij1FFy8FGQDvoD9C2VbOaNhE-uCSZWj245zImbWb90M2btSEoHFblq4GLyNnfQ3K9wkp0lfdzft1JFdfmAxMQhGwUB8NBM=)
8. [readthedocs.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGb6z8B2_EBy0Rr3aGG6nv64zEqqqRqxZGwNPIXuyUCA4usHQUdrfRRTTcb983HFRA65YbFfWwgqRcjkofZ-nrFqHUjl-MT6vZmbh7loqCKVwBoQYdFVXg_YVqS6KQDthR-PnozYvTleRXjZQ==)
9. [hackingtutorials.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH4D5wFKfJ4uFg2Q1f2Zs_7ib9Cg8RMGyF6dZMC0Y2d4KH9VnQrUuF768RQkJ_2WnFmBlhgRULT-DlU5MGLbqHQ7nnp4OapKub9eP6rx52iNAEC7pTuxV3sPFMNI8OFsW3Z4WFaujkGknjBkfI8u3h_uZGx9a9J-NAfNYvmDTiuqg==)
10. [netragard.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHaJMJPXbZhqZV0oJ8dM25tQ_E-ln0nQP0q9HwzmjEKkhrNWCMaNwoEmfIWoxvPXzrq28lKKfxfnuqclddWoTwnsWM-tMlwjdTyLEpiUIVYaLaQ0RpMU93qmj9z6NH3PZfdKR48HWLD-2qsESyUEN6SqvVvTNAaO6Nnlw==)
11. [medium.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHheMSG0YZjU56Mj5Qj5-KVPEH9Ta13rW0Pmu5VoiUaTj-pOAhyaZJPvTu9RazUgZ-LcbqPpSIDyf1ysy77ZzW35gEaOAbWoyl9h535fMu9ZwPn605Z0_iqfwOGMU26WkN9G1vsyAtbILHHpQ7ZGIKwaQTXX7szI9Prj2UJygdinhZ1PNPYwVSLqZ9QjRM8QS_Cn8CNkJx5EzVUutY=)
12. [rapid7.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEzjsPEgw9i67N4Ra9sM2SGzxPoCWtzOQOoj_rCKJF7jmzX2EOtPFVDQRVX2bGNFxSbdJB78qDZjPvjOoByb-bUGp8ik9gw1biroA2ad15RWKHOV6H57dMh0KzTijX25ZA8KO2tAHCtvyOUitn9b8Jtezoc4BqCGuHMN-B0qfwA)
13. [medium.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEk87iYxYrcpFWuu-IM97o1rWb9MXdIy9RwI4cSm92zHA8wSxR8RVTB2oi8018QC2AsyszQowT4ZyjLx65dh-bl0lTMwHmiJ3EMitRs99we2EcP_-dIWJpX_I0dK-I1BkX_npl2dqWmAKFJGlx5wa3VJZl2SnsDLD7Z89Llmi7zKZoIqoZFGdMU4gIXE8rKj8x4p5ca7Goj5izWC3Zlpnkk1xJCrClSHQ==)
14. [python.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFwVdZEw0tVXb6SJPLvNFrj8_QY8FmRW9XC83hn8cYwZY-mLAPx1bbyDjA_eDt1M5JVANFxUPDIUH9gXBvlT-T7qW4rrQemMTC5d4NBxQ4h33m0vHgP0bwdP4Vt9_dThwgFEnh-mgUa5KuzrZJsG6s=)
15. [python.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEYbpSZ2SDyhIXn77deU4xZVMoK2rLdmT6kfOsbKGf4gftkCwEMONFehw6stqDoDnd3kS4hAyTR-w-jqsJSe5-P0ao39e_tKgtkTgMlsXw_evQEJUwTVNiWQSAulabOhlHno5qpn39PvRHd40h0JCk=)
16. [python.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFKVEUxOpQeanz_-JmkobXNtm1mPxMK4k4t-JaEhQPX1WikwkL_DyI3Q7SLNqhBjitJDWfEmgTYxBcGejBGRgNRw9OImKT5zbd5G9AayOE_70teKpRLtF82JcOuP_7B2ORGXaK6h-6IP2yCOZoS8d4gRjg=)
17. [polito.it](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHlKhgv3FZKiJloRdBN46fwOeKqIwPBB9_KRt8s82b6B5THemgDlpAxVwUYinj4uGjyXbdALDSSR65BogvKpYmlh6B3DWUpKOGYakUUj4KUfgTdKpp2KizLt4bxc8qGCHt-ePV7a8xjOb4=)
18. [diva-portal.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE0yDhjVw_w4kAQLSdzafsURdAl0jaPt0QOLG7AofRDsKO0N0exKDfWQaTPMtIVyu_iwtxBxOCn2M180b3I93kzhM_pbm38eSOD1WXsWAl1nJB9j9wRrAcHFa5oEePt2Ugn-L7TG3MHyA_N1xTaJh560hzxzTT9Xg==)
19. [nih.gov](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQH2H7hvV4jSmEA_2KDnJ3jr0Pm95NQ3Om8o_O--FMCASxyKA9GVy5WL5GhFtqVGjkJBYfaCimBWFpNWrW_K0gN_00R-z0Tpte4-LJWsnEPCKVECC6tU_WIYb4jc9fqjsjlSdLHQQj9y)
20. [ctfd.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGLWoVVvuyF5nR3AuK2CzV5SEFiHkBDDyawPsslx-uCebk2ixt6cYXNQB0-ULMuSISpuPdUWpg1tl519sPvHd2eEvs6xhR0fnIqfohKUBqBjjAuZ0qsnKPNcpWlXjDobBM3nQW1XAt0ddfq1K39Z5n7)
21. [ctfd.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGfh5TJJTepsvvRf20PZLXHQCwxqcpxkZ0WPwSyc7QtKluweA398njfljHhKokmNZPVi8A0LoEoTvZSg7U2t4EhEUUjXtRSv4MsqEuass-Qu2qZ7moiPxVUiQRHGPNKWLhR9WnARDmkXkxSnCcR4wZFVC2uOPxItU0=)
22. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGkvQAdDItQsZ0L_J4_VCSJLSXTugV3a0PirFT3a4ssM0XIpqZd6KToxh7EIdilgAkRvrfe6uh1bTNM8XsJ80B0MeXWQMgZKUoIdbPijXLwHuHkJdhwEMSOck3x48Ho2k3qQ_jPRTsTpHsrmf5XWtUVT-tH1g==)
23. [ctfd.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGINl356VUvKxTnYLbp20dqqe-jDBXmOf3tNYKIjht5KgCXS0YTxN8s4-F23loml_N7hJQ_HWOMQf8l1wKoZppo4hpnb21H0OUIqsRUK6oPsEknlL63OjqshM4H3QXhNpn1)
24. [helpnetsecurity.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFJ1dNmnmPqbQD-4jSw9Pfn96giugvXYDyXyeUa1Kvx1jHlRv57Ai29ka6Orfu08MNrOgb00m0wuH-Sn_jzEMpgrdO0jBSyP8HJdy5yhOP7Ku1mAZT9XaMAIZnhC-wDZ78_Basnv0OlTaDmiS2rDKNyN_oxz0NtJOf6sowYLt5ev4im0gCKU7qyKDthC6cdm94xOpVDw8h8dA==)
25. [sredevops.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGyDnn2YDmF0WF_BDh_rZbpjz9uGcL43SvR_oRftl78UShaMZeJ6DhxyRGgB80qZ1velT8af2e40swowwj6mHy6TZi7VU5XjaMXvuR6AF_ori0Y_tZr3Um8abFfRRXWuMmR2vILcNbVcez593HPWc_ub2wKzlIvyBPXYp9J_-ouYBb4duiv90MZCHAKpmhC)
26. [dogesec.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEEU-vh5A_4TdpDhb9KRCQQmJFhXqyuWghP5QJKwURUWBYqfoFuBVLPLFzWnlgHCXLnjjngD-uPkm4gJq0X0lm3K4aApHPJhwmRrqfTudEpvGh5aucSb7e5yCKtodgWHKKEJQJ_dWol7xS9bmjd0MLG)
27. [silentpush.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEoqHouqg43hww36TdJ_tupeHw15vH6ktxTPP5-uB_diLMhfqf9dLlWQSeUv36G9t3c6G82fqarrpwRgkhuKg98ffZmIDqawCTK2ukWZc3K8_Vq9iDX1cqNSE1EWsSs)
28. [opencti.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHmxLqGdGbK42U0lunl3C9tp7W8XQDwK3WMTVgQp6C7wKmxe9JuuzdUnf1VY29tlSYmPaHlqbAiTwa_AGKALuMw4vTfNHH-9kSrkJ6cDGRqAPCM98jx6A6TSq7fvDJ2uLcubfg-6hk=)
29. [mintlify.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGHUPggL0UVhTJKkrf7qC-THYDKO6fuNCJdIMx3ABKfHLT_DGZxaAX6ijtE_SMOcl6rUtUzZseXJypoZJQ9sHdJlsbuIR_UqzHQ4UgVNXYmWjSBv0ZUU-9lMIX5kfYMKNVLLX8vvpe4jAc-1qgCNv3hFlIPu2z-GebgPD-9lLLTJzIF88yHZw2eAcYTWs9tiQ==)
30. [medium.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHOgGm1zoOiuCD9s4U-QPEXzTVTHRMqnjGvcjO29Jj54JccqRXOZ9tG0up10VyC1ZioSQkd4Y1xFyrQz1P7Ycw_Tbl93NOSrl1rwq91H2Twdp1Mlfs6yWDQoFUIi5905xGDqOmceMqWzZWw5qWHGHpTe2ij-zXEawa2a3tBoSzTPCPBg0_z4tKqmf7IqEIcr-PoUV6KxLdhtjNgIDhEfF9kwDt59nmmsN9gcdM8e3OcgI6NnByi)
31. [oneuptime.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHokac4gd_ZLtmfGfgGo_cE7ay4o6L6ZSsEbvP0zRRLKJlNN9FbPqNGJ3V0Be1YOVFYJrTSJUyZxb7VlebZhyrzh3zqCIDyDxhhtnFCgjCiWEl9_5gf5mrNiNJ1LZ7lU7jQEfxReVo_Y-jpTukpepz6_yskOe_CTIRUUXH_qDg8spoXdwMTOBTgt2LW2veD7Dx0yLMhXDuuxA==)
32. [sourceforge.net](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQEuQ1au2Q-Cbu4FHgzYqMcqw_HhUj_gcxRIGDDGu7UJhJIj9N3wSBjKTG2Bs2LXn56AnLCBsCgYRTcg542QyY6S3IVumou25ENgoSdqRimTRHDyAr81h8CmtPxVwgaAbzwJ7PSU--3m)
33. [loginsoft.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGvmqyaK6Utnn6HWJ5WH1mMltr7yfMiT9Q3S-LO05J-OYuvGjc2GeHxSJqBDRiM3BKYF9yKCDw4Wf1Hb3z5qEBkkKDRnMMFu_yqavZHG76_pLwugUYJdbsICIgg23Y3eQOKi7GMPIKAzL2sFHZLMTxFRscIvzQzfd5Mtz68BXoGgjPqUZSkc7QJVesBAPEaac9Uo5bbKyLLTCEdBwjTznVZk2QeSyVR)
34. [jyu.fi](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQECOZJjve3uqqhx5upKhxvh1xrkDF0SuGrPFn5E5A6O6y1qD_lLZnPLa-R8_VHEnVIlWe_oUWOIyx0fhPKnQzIRDfsqdwMqLs0dbkpEwZu1dVs2ruzelViFrn3LkqqDxICDsLU8URLgcFe4KIYHtRRAW4SdM6HI0FKzRkNDdo1W5e4=)
35. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFSxIFxNTcYqlC2YfVqxur2DyAXFQUD-8Pfzd7gzU6RJuB9FfQhnLaDpmVTqkFvP_Puwbx9t54kczukzJgkW1O4kAPVLeduao3YW1d4ruIZrBtZVhKCoWMBkM8dqhFlwUdbhYj9Nl8LVGjBc_E=)
36. [github.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQHl2yvtccFmWowUSLtGC4GY_LzEXSNGKyoW2EkSMtN_0HN0r2VUDoPFCbiogpNurFgT1HgWEwBKOuADRTdgLv2w2lGPLsklfuRk3pBtGwX23HbSSl8BXiPT8pD3V26utpHnSwzrXuZwJSi0jNBQrH-Arfzd)
37. [shadowdragon.io](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGCRfFR6nk_3ozNzFn87Ya2PdJqRLN79jlCYPsSYdf0LSRNkqW06T97Yc_i0Em1VVJbJWe077yU7PEywKfErHrcmbJeKZ9zkFjZBj-x6Ot-emRbp-ZnBLxaWrysh0ZGa5J6lIjo)
38. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFZyKL4Ob5RVwVQckFQwbZdmOd8SFqziZCjgTRKRDQxb0dDfzZd8P4fL4AYADsoBXSMLzgXezcasgx67koDVXir2jzR2UO2nSQ2KP4Pww3FcEnoCl7ClMeHxnz0vvc=)
39. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQGBEMEa7BHaWy02PS2S0ZLSG7TkCKDx8f96xUPnvh_2rY95pJvDGe2BvvOITAMMvyUPtPQQEHSrWERkxd9i1SiUc2DOI4-KiliQDKLfxf0uIMwnbcNeaHclnkn6yLzECbV_p5KtdObvA3ShZdIk)
40. [github.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQFOQ9B5x--evSHwIKBNgZZgm-xENDKkltAyl1PPTfFpy6ddzHRt2bd3jKXkdHPUJq5ey_6mVpdS8ypy5LIOZJ8GW2tBEnddNnwcLhtiTP3ZrcucyIzE_RkosYHxPg==)
41. [threatintel.academy](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQE6vC29stex1y6J7vZYH9MK5fweptpmNngJ5KFfPQo_QcI7cfEAxqsdm_V7bGNYahHaQ2VnFNpQ3Jh0tnqt0yyDvycMeGvMPuRfWQHgr4NHUV-pURTZgVjd3HRP-yFZTpG6yETPeIojdgzcdiO4kC8N1yZJtuwwrM3wcMAZb7LlTYl2Y_-1n6kJi55AND_En72Izdc=)
42. [medium.com](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQExgPItn2cpLErwnOKxpbYZIrqz1jBpRk8hVO-7hzlER2cv9X1o0e5z4kMPe5p4vT330kN9RaUjMzmCLdYdHaXwVOaENNzjqkzUopLsg3pW2KptVTlR4FYp99Jhy7nGNclCGN3MiYwmz_cN4kfSbRemynRkfxMuyYC-q_KEF6YGtLM=)
