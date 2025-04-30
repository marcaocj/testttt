// Player - Extensão de CharacterStats para o jogador
using UnityEngine;
using System;

public class Player : CharacterStats
{
    [SerializeField] private int statPoints = 0;
    
    public event Action<int> OnStatPointsChanged;
    public event Action<int> OnExperienceChanged;
    
    // Cache de componentes
    private PlayerMovement movement;
    private PlayerCombat combat;
    
    protected override void Start()
    {
        base.Start();
        
        // Inicializa componentes relacionados
        movement = GetComponent<PlayerMovement>();
        combat = GetComponent<PlayerCombat>();
    }
    
    public override void CalculateDerivedStats()
    {
        // Fórmulas específicas para o jogador (pode ser igual ou diferente do monstro)
        base.CalculateDerivedStats();
        
        // Modificações adicionais baseadas em equipamentos ou habilidades podem ser adicionadas aqui
    }
    
    protected override void LevelUp()
    {
        base.LevelUp();
        
        // Concede pontos de status ao subir de nível
        statPoints += 3;
        OnStatPointsChanged?.Invoke(statPoints);
        
        // Recupera HP e SP
        derivedStats.CurrentHP = derivedStats.MaxHP;
        derivedStats.CurrentSP = derivedStats.MaxSP;
        
        // Para corrigir o erro: Não podemos invocar diretamente eventos da classe base
        // Usamos ModifyHP e ModifySP, que vão disparar os eventos correspondentes internamente
        ModifyHP(0); // Isso vai disparar o evento OnHPChanged com os valores atuais
        ModifySP(0); // Isso vai disparar o evento OnSPChanged com os valores atuais
        
        // Efeito visual ou de som para subida de nível pode ser adicionado aqui
    }
    
    public override void AddExperience(int amount)
    {
        base.AddExperience(amount);
        
        // Notifica sobre mudança de experiência
        OnExperienceChanged?.Invoke(experiencePoints);
    }
	
    public void AddStatPoint(string statName)
    {
        if (statPoints <= 0) return;
        
        switch (statName)
        {
            case "STR":
                baseStats.STR++;
                break;
            case "AGI":
                baseStats.AGI++;
                break;
            case "VIT":
                baseStats.VIT++;
                break;
            case "INT":
                baseStats.INT++;
                break;
            case "DEX":
                baseStats.DEX++;
                break;
            case "LUK":
                baseStats.LUK++;
                break;
            default:
                return;
        }
        
        // Decrementa os pontos disponíveis
        statPoints--;
        OnStatPointsChanged?.Invoke(statPoints);
        
        // Recalcula os status derivados
        CalculateDerivedStats();
    }
    
    protected override void Die()
    {
        base.Die();
        
        // Implementação específica da morte do jogador
        Debug.Log("Jogador morreu");
        
        // Pode adicionar lógica de penalidade por morte aqui
        
        // Reaparecer em um ponto seguro ou mostrar tela de morte
        EventManager.Instance.TriggerEvent("PlayerDied");
    }
    
    // Getters adicionais específicos para o jogador
    public int GetStatPoints() => statPoints;
    public int GetCurrentExperience() => experiencePoints;
    public int GetExperienceToNextLevel() => experienceToNextLevel;
}
// PlayerCombat - Gerencia o combate do jogador
using UnityEngine;
using System.Collections;

public class PlayerCombat : MonoBehaviour
{
    [SerializeField] private PlayerMovement movement;
    [SerializeField] private Animator animator;
    [SerializeField] private float attackCooldown = 1.5f;
    
    private Player playerStats;
    private Monster currentTarget; 
    private bool isAttacking = false;
    private float lastAttackTime = 0f;
    
    private static readonly int AttackTrigger = Animator.StringToHash("Attack");
    
    private void Start()
    {
        playerStats = GetComponent<Player>();
        
        if (movement == null)
        {
            movement = GetComponent<PlayerMovement>();
        }
        
        if (animator == null)
        {
            animator = GetComponent<Animator>();
        }
    }
    
    public void AttackTarget(Monster target)
    {
        if (isAttacking) return;
        
        currentTarget = target;
        
        // Verifica se o alvo está ao alcance
        if (CombatManager.Instance.IsInAttackRange(transform, target.transform))
        {
            PerformAttack();
        }
        else
        {
            // Move até o alvo e depois ataca
            movement.MoveTo(target.transform.position);
            StartCoroutine(MoveAndAttack());
        }
    }
    
    private IEnumerator MoveAndAttack()
    {
        // Espera até estar perto o suficiente ou o alvo morrer
        while (currentTarget != null && 
               currentTarget.gameObject.activeInHierarchy && 
               !CombatManager.Instance.IsInAttackRange(transform, currentTarget.transform))
        {
            // Atualiza o destino caso o alvo se mova
            movement.MoveTo(currentTarget.transform.position);
            yield return null;
        }
        
        // Verifica se o alvo ainda está vivo
        if (currentTarget != null && currentTarget.gameObject.activeInHierarchy)
        {
            // Para de se mover
            movement.StopMoving();
            
            // Realiza o ataque
            PerformAttack();
        }
    }
    
    private void PerformAttack()
    {
        // Verifica o cooldown
        if (Time.time - lastAttackTime < GetAttackCooldown())
        {
            return;
        }
        
        // Olha para o alvo
        LookAtTarget();
        
        // Atualiza o tempo do último ataque
        lastAttackTime = Time.time;
        
        // Inicia a animação de ataque
        animator.SetTrigger(AttackTrigger);
        
        // Marca que está atacando para evitar interrupções
        isAttacking = true;
        
        // Inicia a coroutine que lidará com o timing do dano
        StartCoroutine(ApplyDamageAfterAnimation());
    }
    
    private IEnumerator ApplyDamageAfterAnimation()
    {
        // Espera um tempo para sincronizar com a animação
        yield return new WaitForSeconds(0.5f);
        
        // Verifica se o alvo ainda está vivo e ao alcance
        if (currentTarget != null && 
            currentTarget.gameObject.activeInHierarchy && 
            CombatManager.Instance.IsInAttackRange(transform, currentTarget.transform))
        {
            // Calcula o dano
            int damage = CombatManager.Instance.CalculateDamage(playerStats, currentTarget);
            
            // Aplica o dano
            currentTarget.TakeDamage(damage, gameObject);
            
            // Dispara evento de dano causado
            EventManager.Instance.TriggerEvent("PlayerDamageCaused", damage);
        }
        
        // Espera o fim da animação
        yield return new WaitForSeconds(0.5f);
        
        // Marca que terminou o ataque
        isAttacking = false;
        
        // Se o alvo ainda estiver vivo e ao alcance, continua atacando
        if (currentTarget != null && 
            currentTarget.gameObject.activeInHierarchy && 
            CombatManager.Instance.IsInAttackRange(transform, currentTarget.transform))
        {
            PerformAttack();
        }
    }
    
    private void LookAtTarget()
    {
        if (currentTarget == null) return;
        
        Vector3 direction = currentTarget.transform.position - transform.position;
        direction.y = 0; // Ignora diferença de altura
        
        if (direction != Vector3.zero)
        {
            transform.rotation = Quaternion.LookRotation(direction);
        }
    }
    
    private float GetAttackCooldown()
    {
        DerivedStats stats = playerStats.GetDerivedStats();
        
        // Calcula o cooldown baseado na velocidade de ataque (ASPD)
        // Quanto maior o ASPD, menor o cooldown
        float aspd = stats.ASPD;
        
        // Fórmula similar ao Ragnarok Online
        float cooldown = Mathf.Max(attackCooldown * (200f / aspd), 0.5f);
        
        return cooldown;
    }
}
// PlayerInput - Gerencia as entradas do usuário
using UnityEngine;

public class PlayerInput : MonoBehaviour
{
    [SerializeField] private PlayerMovement movement;
    [SerializeField] private PlayerCombat combat;
    [SerializeField] private float interactionDistance = 2f;
    
    private Camera mainCamera;
    
    private void Start()
    {
        mainCamera = Camera.main;
    }
    
    private void Update()
    {
        HandleMouseInput();
        HandleCameraInput();
    }
    
    private void HandleMouseInput()
    {
        if (Input.GetMouseButtonDown(0)) // Botão esquerdo do mouse
        {
            Ray ray = mainCamera.ScreenPointToRay(Input.mousePosition);
            RaycastHit hit;
            
            if (Physics.Raycast(ray, out hit))
            {
                // Verifica que tipo de objeto foi clicado
                if (hit.collider.CompareTag("Ground"))
                {
                    // Mover o jogador
                    movement.MoveTo(hit.point);
                }
                else if (hit.collider.CompareTag("Monster"))
                {
                    // Ataca o monstro
                    Monster monster = hit.collider.GetComponent<Monster>();
                    if (monster != null)
                    {
                        combat.AttackTarget(monster);
                    }
                }
                else if (hit.collider.CompareTag("Item"))
                {
                    // Pega o item
                    GroundItem item = hit.collider.GetComponent<GroundItem>();
                    if (item != null)
                    {
                        // Calcula a distância
                        float distance = Vector3.Distance(transform.position, hit.point);
                        if (distance <= interactionDistance)
                        {
                            // Pega o item imediatamente
                            item.PickUp();
                        }
                        else
                        {
                            // Move-se até o item primeiro
                            movement.MoveTo(hit.point);
                            // Implementação de pickup após chegar ao item será feita depois
                        }
                    }
                }
            }
        }
    }
    
    private void HandleCameraInput()
    {
        // Implementação do controle de câmera será feita em outro componente
    }
}
// PlayerManager - Gerencia o jogador e suas ações
using UnityEngine;

public class PlayerManager : MonoBehaviour
{
    public static PlayerManager Instance { get; private set; }
    
    [SerializeField] private GameObject playerPrefab;
    [SerializeField] private Transform spawnPoint;
    
    private Player playerInstance;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public void Initialize()
    {
        // Verifica se o jogador já existe na cena
        playerInstance = GameObject.FindGameObjectWithTag("Player")?.GetComponent<Player>();
        
        if (playerInstance == null && playerPrefab != null)
        {
            // Instancia o jogador no ponto de spawn
            Vector3 spawnPosition = (spawnPoint != null) ? spawnPoint.position : Vector3.zero;
            GameObject playerObj = Instantiate(playerPrefab, spawnPosition, Quaternion.identity);
            playerInstance = playerObj.GetComponent<Player>();
            
            if (playerInstance == null)
            {
                Debug.LogError("O prefab do jogador não contém o componente Player");
            }
        }
        
        // Registra eventos do jogador
        RegisterPlayerEvents();
        
        Debug.Log("Player system initialized");
    }
    
    private void RegisterPlayerEvents()
    {
        if (playerInstance != null)
        {
            // Registra para receber eventos de morte do jogador
            EventManager.Instance.AddListener("PlayerDied", OnPlayerDied);
        }
    }
    
    private void OnPlayerDied(object data)
    {
        // Lógica de respawn ou game over
        Debug.Log("Jogador morreu - iniciando respawn");
        RespawnPlayer();
    }
    
    public void RespawnPlayer()
    {
        if (playerInstance != null)
        {
            // Reposiciona o jogador
            Vector3 respawnPosition = (spawnPoint != null) ? spawnPoint.position : Vector3.zero;
            playerInstance.transform.position = respawnPosition;
            
            // Recupera vida e status
            DerivedStats stats = playerInstance.GetDerivedStats();
            stats.CurrentHP = stats.MaxHP;
            stats.CurrentSP = stats.MaxSP;
            
            // Notifica mudanças
            playerInstance.SendMessage("OnHPChanged", new object[] { stats.CurrentHP, stats.MaxHP }, SendMessageOptions.DontRequireReceiver);
            playerInstance.SendMessage("OnSPChanged", new object[] { stats.CurrentSP, stats.MaxSP }, SendMessageOptions.DontRequireReceiver);
        }
    }
    
    public Player GetPlayer()
    {
        return playerInstance;
    }
    
    private void OnDestroy()
    {
        // Remove listeners
        if (EventManager.Instance != null)
        {
            EventManager.Instance.RemoveListener("PlayerDied", OnPlayerDied);
        }
    }
}
// PlayerMovement - Controla a movimentação do personagem
using UnityEngine;
using UnityEngine.AI;

public class PlayerMovement : MonoBehaviour
{
    [SerializeField] private NavMeshAgent agent;
    [SerializeField] private Animator animator;
    [SerializeField] private float stoppingDistance = 0.1f;
    
    private Camera mainCamera;
    private static readonly int IsMovingParam = Animator.StringToHash("IsMoving");
    
    private void Start()
    {
        mainCamera = Camera.main;
        agent.stoppingDistance = stoppingDistance;
    }
    
    private void Update()
    {
        // Atualiza o parâmetro de animação com base no movimento
        animator.SetBool(IsMovingParam, agent.velocity.magnitude > 0.1f);
    }
    
    public void MoveTo(Vector3 position)
    {
        agent.SetDestination(position);
    }
    
    public void StopMoving()
    {
        agent.ResetPath();
    }
}
// UIManager - Gerencia a interface do usuário
using UnityEngine;
using UnityEngine.UI;
using TMPro;

public class UIManager : MonoBehaviour
{
    public static UIManager Instance { get; private set; }
    
    [Header("Status do Jogador")]
    [SerializeField] private Slider hpSlider;
    [SerializeField] private Slider spSlider;
    [SerializeField] private TextMeshProUGUI hpText;
    [SerializeField] private TextMeshProUGUI spText;
    [SerializeField] private TextMeshProUGUI levelText;
    [SerializeField] private Slider expSlider;
    [SerializeField] private TextMeshProUGUI expText;
    
    [Header("Painel de Status")]
    [SerializeField] private GameObject statusPanel;
    [SerializeField] private TextMeshProUGUI strText;
    [SerializeField] private TextMeshProUGUI agiText;
    [SerializeField] private TextMeshProUGUI vitText;
    [SerializeField] private TextMeshProUGUI intText;
    [SerializeField] private TextMeshProUGUI dexText;
    [SerializeField] private TextMeshProUGUI lukText;
    [SerializeField] private TextMeshProUGUI statPointsText;
    [SerializeField] private Button[] statButtons;
    
    [Header("Alvo")]
    [SerializeField] private GameObject targetPanel;
    [SerializeField] private TextMeshProUGUI targetNameText;
    [SerializeField] private Slider targetHpSlider;
    [SerializeField] private TextMeshProUGUI targetHpText;
    
    private Player player;
    private Monster currentTarget;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    private void Start()
    {
        player = GameObject.FindGameObjectWithTag("Player").GetComponent<Player>();
        
        if (player != null)
        {
            // Registra os callbacks para as mudanças de status
            player.OnHPChanged += UpdatePlayerHP;
            player.OnSPChanged += UpdatePlayerSP;
            player.OnLevelUp += UpdatePlayerLevel;
            player.OnStatPointsChanged += UpdateStatPoints;
            
            // Inicializa a UI com os valores atuais
            UpdateAllStats();
        }
        
        // Configura os botões de status
        SetupStatButtons();
        
        // Esconde o painel de alvo inicialmente
        if (targetPanel != null)
        {
            targetPanel.SetActive(false);
        }
    }
    
    private void OnDestroy()
    {
        if (player != null)
        {
            // Remove os callbacks
            player.OnHPChanged -= UpdatePlayerHP;
            player.OnSPChanged -= UpdatePlayerSP;
            player.OnLevelUp -= UpdatePlayerLevel;
            player.OnStatPointsChanged -= UpdateStatPoints;
        }
    }
    
    public void UpdateAllStats()
    {
        if (player == null) return;
        
        DerivedStats stats = player.GetDerivedStats();
        
        // Atualiza HP/SP
        UpdatePlayerHP(stats.CurrentHP, stats.MaxHP);
        UpdatePlayerSP(stats.CurrentSP, stats.MaxSP);
        
        // Atualiza nível
        UpdatePlayerLevel(player.GetLevel());
        
        // Atualiza experiência
        // Aqui você precisaria de métodos para obter a experiência atual e máxima
        // UpdatePlayerExp(player.GetCurrentExp(), player.GetExpToNextLevel());
        
        // Atualiza estatísticas base
        UpdateBaseStats();
    }
    
    private void UpdatePlayerHP(int current, int max)
    {
        if (hpSlider != null)
        {
            hpSlider.maxValue = max;
            hpSlider.value = current;
        }
        
        if (hpText != null)
        {
            hpText.text = $"{current}/{max}";
        }
    }
    
    private void UpdatePlayerSP(int current, int max)
    {
        if (spSlider != null)
        {
            spSlider.maxValue = max;
            spSlider.value = current;
        }
        
        if (spText != null)
        {
            spText.text = $"{current}/{max}";
        }
    }
    
    private void UpdatePlayerLevel(int level)
    {
        if (levelText != null)
        {
            levelText.text = $"Lv. {level}";
        }
    }
    
    private void UpdatePlayerExp(int current, int max)
    {
        if (expSlider != null)
        {
            expSlider.maxValue = max;
            expSlider.value = current;
        }
        
        if (expText != null)
        {
            float percentage = (float)current / max * 100f;
            expText.text = $"{percentage:F1}%";
        }
    }
    
    private void UpdateBaseStats()
    {
        if (player == null) return;
        
        BaseStats baseStats = player.GetBaseStats();
        
        if (strText != null) strText.text = baseStats.STR.ToString();
        if (agiText != null) agiText.text = baseStats.AGI.ToString();
        if (vitText != null) vitText.text = baseStats.VIT.ToString();
        if (intText != null) intText.text = baseStats.INT.ToString();
        if (dexText != null) dexText.text = baseStats.DEX.ToString();
        if (lukText != null) lukText.text = baseStats.LUK.ToString();
    }
    
    private void UpdateStatPoints(int points)
    {
        if (statPointsText != null)
        {
            statPointsText.text = $"Pontos: {points}";
        }
        
        // Ativa/desativa os botões de status baseado nos pontos disponíveis
        bool buttonsInteractable = points > 0;
        foreach (Button button in statButtons)
        {
            button.interactable = buttonsInteractable;
        }
    }
    
    private void SetupStatButtons()
    {
        // Configura os listeners para os botões de status
        if (statButtons == null || statButtons.Length == 0) return;
        
        string[] statNames = { "STR", "AGI", "VIT", "INT", "DEX", "LUK" };
        
        for (int i = 0; i < statButtons.Length && i < statNames.Length; i++)
        {
            string statName = statNames[i];
            statButtons[i].onClick.AddListener(() => AddStatPoint(statName));
        }
    }
    
    private void AddStatPoint(string statName)
    {
        if (player != null)
        {
            player.AddStatPoint(statName);
            UpdateBaseStats();
        }
    }
    
    public void SetTarget(Monster target)
    {
        if (target == null)
        {
            // Remove o alvo atual
            currentTarget = null;
            
            if (targetPanel != null)
            {
                targetPanel.SetActive(false);
            }
            
            return;
        }
        
        // Define o novo alvo
        currentTarget = target;
        
        // Mostra o painel de alvo
        if (targetPanel != null)
        {
            targetPanel.SetActive(true);
            
            if (targetNameText != null)
            {
                targetNameText.text = target.gameObject.name.Replace("(Clone)", "").Trim();
            }
            
            // Registra para eventos de HP do alvo
            target.OnHPChanged += UpdateTargetHP;
            
            // Atualiza a barra de HP do alvo
            DerivedStats stats = target.GetDerivedStats();
            UpdateTargetHP(stats.CurrentHP, stats.MaxHP);
        }
    }
    
    private void UpdateTargetHP(int current, int max)
    {
        if (targetHpSlider != null)
        {
            targetHpSlider.maxValue = max;
            targetHpSlider.value = current;
        }
        
        if (targetHpText != null)
        {
            targetHpText.text = $"{current}/{max}";
        }
        
        // Se o HP chegar a zero, remove o alvo
        if (current <= 0)
        {
            // Remove o callback para evitar leak de memória
            if (currentTarget != null)
            {
                currentTarget.OnHPChanged -= UpdateTargetHP;
            }
            
            // Esconde o painel após um delay
            Invoke("HideTargetPanel", 2f);
        }
    }
    
    private void HideTargetPanel()
    {
        if (targetPanel != null)
        {
            targetPanel.SetActive(false);
        }
    }
    
    public void ToggleStatusPanel()
    {
        if (statusPanel != null)
        {
            statusPanel.SetActive(!statusPanel.activeSelf);
            
            if (statusPanel.activeSelf)
            {
                UpdateAllStats();
            }
        }
    }
}
// MonsterManager - Gerencia o spawn e respawn de monstros
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class MonsterManager : MonoBehaviour
{
    public static MonsterManager Instance { get; private set; }
    
    [System.Serializable]
    public class MonsterSpawnInfo
    {
        public GameObject monsterPrefab;
        public Transform spawnArea;
        public int count = 5;
        public float respawnTime = 30f;
    }
    
    [SerializeField] private List<MonsterSpawnInfo> monsterSpawns = new List<MonsterSpawnInfo>();
    
    private Dictionary<GameObject, Coroutine> respawnCoroutines = new Dictionary<GameObject, Coroutine>();
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public void Initialize()
    {
        // Spawn inicial de todos os monstros
        foreach (MonsterSpawnInfo spawnInfo in monsterSpawns)
        {
            SpawnMonstersInArea(spawnInfo);
        }
    }
    
    private void SpawnMonstersInArea(MonsterSpawnInfo spawnInfo)
    {
        if (spawnInfo.spawnArea == null || spawnInfo.monsterPrefab == null)
        {
            Debug.LogError("Invalid spawn info");
            return;
        }
        
        // Obtém os bounds da área de spawn
        Renderer renderer = spawnInfo.spawnArea.GetComponent<Renderer>();
        if (renderer == null)
        {
            Debug.LogError("Spawn area needs a renderer component");
            return;
        }
        
        Bounds bounds = renderer.bounds;
        
        for (int i = 0; i < spawnInfo.count; i++)
        {
            // Encontra uma posição válida dentro da área
            Vector3 spawnPosition = GetRandomPointInBounds(bounds);
            
            // Verifica se a posição está no NavMesh
            UnityEngine.AI.NavMeshHit hit;
            if (UnityEngine.AI.NavMesh.SamplePosition(spawnPosition, out hit, 10f, UnityEngine.AI.NavMesh.AllAreas))
            {
                // Instancia o monstro
                GameObject monster = Instantiate(spawnInfo.monsterPrefab, hit.position, Quaternion.identity);
                
                // Configura o ponto de spawn
                Monster monsterComponent = monster.GetComponent<Monster>();
                if (monsterComponent != null)
                {
                    Transform spawnPoint = new GameObject($"{monster.name}_SpawnPoint").transform;
                    spawnPoint.position = hit.position;
                    
                    // Aqui você precisaria ter um método para definir o spawn point no Monster
                    // monsterComponent.SetSpawnPoint(spawnPoint);
                }
            }
        }
    }
    
    private Vector3 GetRandomPointInBounds(Bounds bounds)
    {
        return new Vector3(
            Random.Range(bounds.min.x, bounds.max.x),
            bounds.min.y, // Usa a altura mínima
            Random.Range(bounds.min.z, bounds.max.z)
        );
    }
    
    public void ScheduleRespawn(GameObject monster, Vector3 spawnPosition)
    {
        // Encontra o spawn info correspondente
        MonsterSpawnInfo spawnInfo = null;
        foreach (MonsterSpawnInfo info in monsterSpawns)
        {
            if (info.monsterPrefab.name == monster.name.Replace("(Clone)", "").Trim())
            {
                spawnInfo = info;
                break;
            }
        }
        
        if (spawnInfo == null)
        {
            Debug.LogWarning($"No spawn info found for monster: {monster.name}");
            return;
        }
        
        // Inicia a coroutine de respawn
        if (respawnCoroutines.ContainsKey(monster))
        {
            StopCoroutine(respawnCoroutines[monster]);
        }
        
        respawnCoroutines[monster] = StartCoroutine(RespawnMonster(monster, spawnPosition, spawnInfo.respawnTime));
    }
    
    private IEnumerator RespawnMonster(GameObject monster, Vector3 spawnPosition, float respawnTime)
    {
        yield return new WaitForSeconds(respawnTime);
        
        // Reativa o monstro na posição original
        monster.transform.position = spawnPosition;
        
        // Reativa componentes
        monster.SetActive(true);
        monster.GetComponent<Collider>().enabled = true;
        monster.GetComponent<UnityEngine.AI.NavMeshAgent>().enabled = true;
		
        // Reinicia o HP do monstro
        Monster monsterComponent = monster.GetComponent<Monster>();
        if (monsterComponent != null)
        {
            // Reset dos status do monstro
            DerivedStats stats = monsterComponent.GetDerivedStats();
            stats.CurrentHP = stats.MaxHP;
            
            // Notifica sobre a mudança de HP
            monsterComponent.SendMessage("OnHPChanged", new object[] { stats.CurrentHP, stats.MaxHP }, SendMessageOptions.DontRequireReceiver);
        }
        
        // Remove da lista de coroutines
        respawnCoroutines.Remove(monster);
    }
}
// Monster - Extensão de CharacterStats para monstros
using UnityEngine;
using UnityEngine.AI;
using System.Collections;

public enum MonsterState
{
    Idle,
    Patrol,
    Chase,
    Attack,
    Return,
    Die
}

public class Monster : CharacterStats
{
    [SerializeField] private MonsterState currentState = MonsterState.Idle;
    [SerializeField] private float aggroRange = 5f;
    [SerializeField] private float attackRange = 1.5f;
    [SerializeField] private float attackCooldown = 2f;
    [SerializeField] private int experienceReward = 10;
    [SerializeField] private Transform spawnPoint;
    [SerializeField] private float maxChaseDistance = 15f;
    
    private NavMeshAgent agent;
    private Animator animator;
    private Transform player;
    private float lastAttackTime;
    private bool canAttack = true;
    
    private static readonly int StateParam = Animator.StringToHash("State");
    private static readonly int AttackTrigger = Animator.StringToHash("Attack");
    private static readonly int DamageTrigger = Animator.StringToHash("TakeDamage");
    private static readonly int DeathTrigger = Animator.StringToHash("Die");
    
    protected override void Start()
    {
        base.Start();
        
        agent = GetComponent<NavMeshAgent>();
        animator = GetComponent<Animator>();
        player = GameObject.FindGameObjectWithTag("Player").transform;
        
        if (spawnPoint == null)
        {
            spawnPoint = new GameObject($"{gameObject.name}_SpawnPoint").transform;
            spawnPoint.position = transform.position;
        }
        
        // Inicia a máquina de estados
        StartCoroutine(StateMachine());
    }
    
    private IEnumerator StateMachine()
    {
        while (true)
        {
            yield return StartCoroutine(currentState.ToString());
        }
    }
    
    private IEnumerator Idle()
    {
        // Atualiza o animator
        animator.SetInteger(StateParam, (int)MonsterState.Idle);
        
        while (currentState == MonsterState.Idle)
        {
            // Verifica se o jogador está no raio de aggro
            if (IsPlayerInRange(aggroRange))
            {
                currentState = MonsterState.Chase;
                yield break;
            }
            
            yield return null;
        }
    }
    
    private IEnumerator Patrol()
    {
        // Implementação da patrulha (pode ser expandida depois)
        currentState = MonsterState.Idle;
        yield break;
    }
    
    private IEnumerator Chase()
    {
        // Atualiza o animator
        animator.SetInteger(StateParam, (int)MonsterState.Chase);
        
        while (currentState == MonsterState.Chase)
        {
            // Move em direção ao jogador
            agent.SetDestination(player.position);
            
            // Verifica se está perto o suficiente para atacar
            if (IsPlayerInRange(attackRange))
            {
                currentState = MonsterState.Attack;
                yield break;
            }
            
            // Verifica se está muito longe do ponto de spawn
            if (Vector3.Distance(transform.position, spawnPoint.position) > maxChaseDistance)
            {
                currentState = MonsterState.Return;
                yield break;
            }
            
            // Verifica se o jogador saiu do raio de aggro
            if (!IsPlayerInRange(aggroRange * 1.5f)) // Um pouco maior para não ficar alternando estados
            {
                currentState = MonsterState.Return;
                yield break;
            }
            
            yield return null;
        }
    }
    
    private IEnumerator Attack()
    {
        while (currentState == MonsterState.Attack)
        {
            // Verifica se pode atacar
            if (canAttack && Time.time - lastAttackTime >= attackCooldown)
            {
                // Vira em direção ao jogador
                LookAtPlayer();
                
                // Atualiza o animator
                animator.SetTrigger(AttackTrigger);
                
                // Realiza o ataque após um pequeno delay (para sincronizar com a animação)
                lastAttackTime = Time.time;
                canAttack = false;
                
                // Delay para sincronizar com a animação
                yield return new WaitForSeconds(0.5f);
                
                // Calcula e aplica o dano
                if (IsPlayerInRange(attackRange))
                {
                    PerformAttack();
                }
                
                // Permite atacar novamente após o cooldown
                canAttack = true;
            }
            
            // Verifica se o jogador saiu do alcance de ataque
            if (!IsPlayerInRange(attackRange))
            {
                currentState = MonsterState.Chase;
                yield break;
            }
            
            yield return null;
        }
    }
    
    private IEnumerator Return()
    {
        // Atualiza o animator
        animator.SetInteger(StateParam, (int)MonsterState.Idle);
        
        // Retorna ao ponto de spawn
        agent.SetDestination(spawnPoint.position);
        
        while (currentState == MonsterState.Return)
        {
            // Verifica se chegou ao ponto de spawn
            if (Vector3.Distance(transform.position, spawnPoint.position) < 0.5f)
            {
                // Recupera totalmente o HP ao retornar
                // FIX: Using ModifyHP instead of direct event invocation
                int hpToRestore = derivedStats.MaxHP - derivedStats.CurrentHP;
                if (hpToRestore > 0)
                {
                    ModifyHP(hpToRestore);
                }
                
                currentState = MonsterState.Idle;
                yield break;
            }
            
            // Se o jogador entrar no raio de aggro novamente durante o retorno
            if (IsPlayerInRange(aggroRange) && 
                Vector3.Distance(transform.position, spawnPoint.position) < maxChaseDistance * 0.5f)
            {
                currentState = MonsterState.Chase;
                yield break;
            }
            
            yield return null;
        }
    }
    
    private IEnumerator HandleDeath()
    {
        // Atualiza o animator
        animator.SetTrigger(DeathTrigger);
        
        // Desativa o NavMeshAgent
        agent.enabled = false;
        
        // Aguarda o fim da animação
        yield return new WaitForSeconds(2f);
        
        // Concede experiência ao jogador
        if (player != null)
        {
            Player playerComponent = player.GetComponent<Player>();
            if (playerComponent != null)
            {
                playerComponent.AddExperience(experienceReward);
            }
        }
        
        // Implementação do drop de itens aqui
        
        // Destroi o monstro (ou desativa para pool)
        gameObject.SetActive(false);
        
        // Respawn após um tempo
        MonsterManager.Instance.ScheduleRespawn(gameObject, spawnPoint.position);
    }
    
    public void TakeDamage(int damage, GameObject attacker)
    {
        ModifyHP(-damage);
        
        // Atualiza o animator
        animator.SetTrigger(DamageTrigger);
        
        // Se for atacado pelo jogador, muda para estado de perseguição
        if (attacker.CompareTag("Player") && currentState != MonsterState.Attack && currentState != MonsterState.Die)
        {
            currentState = MonsterState.Chase;
            player = attacker.transform;
        }
    }
    
    private void PerformAttack()
    {
        // Obtém o componente Player
        Player playerComponent = player.GetComponent<Player>();
        if (playerComponent != null)
        {
            // Calcula o dano baseado nos status
            int damage = CalculateDamage(playerComponent);
            
            // Aplica o dano através do EventManager
            EventManager.Instance.TriggerEvent("PlayerDamaged", damage);
        }
    }
    
    protected override void Die()
    {
        base.Die();
        
        // Muda para o estado de morte
        currentState = MonsterState.Die;
        
        // Desativa colisões
        GetComponent<Collider>().enabled = false;
        
        // Inicia a coroutine de morte
        StartCoroutine(HandleDeath());
    }
    
    private int CalculateDamage(Player target)
    {
        // Cálculo similar ao rAthena
        DerivedStats playerStats = target.GetDerivedStats();
        
        int baseDamage = derivedStats.ATK;
        int defense = playerStats.DEF;
        
        // Fórmula simplificada de dano
        int finalDamage = Mathf.Max(baseDamage - defense, 1);
        
        // Chance de crítico
        if (Random.Range(0f, 100f) < derivedStats.CRIT)
        {
            finalDamage = (int)(finalDamage * 1.5f);
        }
        
        return finalDamage;
    }
    
    private bool IsPlayerInRange(float range)
    {
        if (player == null) return false;
        return Vector3.Distance(transform.position, player.position) <= range;
    }
    
    private void LookAtPlayer()
    {
        if (player == null) return;
        
        Vector3 direction = player.position - transform.position;
        direction.y = 0; // Ignora diferença de altura
        
        if (direction != Vector3.zero)
        {
            transform.rotation = Quaternion.LookRotation(direction);
        }
    }
}
// ItemDatabase - Contém informações sobre todos os itens do jogo
using UnityEngine;
using System.Collections.Generic;

[System.Serializable]
public class ItemData
{
    public int id;
    public string name;
    public string description;
    public Sprite icon;
    public bool isStackable = true;
    public int maxStackSize = 99;
    public ItemType type;
    public int value; // Valor para venda/compra
    
    // Estatísticas para equipamentos
    public int atkBonus;
    public int defBonus;
    public int matkBonus;
    public int mdefBonus;
    public int hpBonus;
    public int spBonus;
    
    // Estatísticas para consumíveis
    public int hpRestore;
    public int spRestore;
}

public enum ItemType
{
    None,
    Weapon,
    Armor,
    Accessory,
    Consumable,
    Material,
    Quest
}

public class ItemDatabase : MonoBehaviour
{
    public static ItemDatabase Instance { get; private set; }
    
    [SerializeField] private List<ItemData> items = new List<ItemData>();
    
    private Dictionary<int, ItemData> itemDictionary = new Dictionary<int, ItemData>();
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
        
        // Constrói o dicionário para acesso rápido
        BuildDictionary();
    }
    
    private void BuildDictionary()
    {
        itemDictionary.Clear();
        
        foreach (ItemData item in items)
        {
            if (!itemDictionary.ContainsKey(item.id))
            {
                itemDictionary.Add(item.id, item);
            }
            else
            {
                Debug.LogError($"Item com ID duplicado: {item.id} - {item.name}");
            }
        }
        
        Debug.Log($"ItemDatabase inicializado com {itemDictionary.Count} itens");
    }
    
    public ItemData GetItem(int id)
    {
        if (itemDictionary.ContainsKey(id))
        {
            return itemDictionary[id];
        }
        
        Debug.LogWarning($"Item com ID {id} não encontrado");
        return null;
    }
    
    public bool IsItemStackable(int id)
    {
        ItemData item = GetItem(id);
        return item != null && item.isStackable;
    }
    
    public int GetMaxStackSize(int id)
    {
        ItemData item = GetItem(id);
        return item != null ? item.maxStackSize : 1;
    }
}
// InventoryManager - Gerencia o inventário do jogador
using UnityEngine;
using System.Collections.Generic;

public class InventoryManager : MonoBehaviour
{
    public static InventoryManager Instance { get; private set; }
    
    [System.Serializable]
    public class InventorySlot
    {
        public int itemId;
        public int amount;
        public bool isEmpty => itemId <= 0 || amount <= 0;
    }
    
    [SerializeField] private int inventorySize = 100;
    [SerializeField] private List<InventorySlot> slots = new List<InventorySlot>();
    
    public event System.Action OnInventoryChanged;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
        
        // Inicializa o inventário
        InitializeInventory();
    }
    
    private void InitializeInventory()
    {
        slots.Clear();
        
        for (int i = 0; i < inventorySize; i++)
        {
            slots.Add(new InventorySlot());
        }
    }
    
    public bool AddItem(int itemId, int amount = 1)
    {
        if (itemId <= 0 || amount <= 0) return false;
        
        // Verifica se o item é empilhável
        bool isStackable = ItemDatabase.Instance.IsItemStackable(itemId);
        
        if (isStackable)
        {
            // Procura por slots que já contenham o item
            for (int i = 0; i < slots.Count; i++)
            {
                if (slots[i].itemId == itemId)
                {
                    slots[i].amount += amount;
                    OnInventoryChanged?.Invoke();
                    return true;
                }
            }
        }
        
        // Se não encontrou um slot com o mesmo item ou o item não é empilhável,
        // procura por um slot vazio
        for (int i = 0; i < slots.Count; i++)
        {
            if (slots[i].isEmpty)
            {
                slots[i].itemId = itemId;
                slots[i].amount = amount;
                OnInventoryChanged?.Invoke();
                return true;
            }
        }
        
        // Inventário cheio
        return false;
    }
    
    public bool RemoveItem(int itemId, int amount = 1)
    {
        if (itemId <= 0 || amount <= 0) return false;
        
        // Procura pelo item no inventário
        for (int i = 0; i < slots.Count; i++)
        {
            if (slots[i].itemId == itemId)
            {
                if (slots[i].amount > amount)
                {
                    slots[i].amount -= amount;
                }
                else
                {
                    // Remove o item completamente
                    slots[i].itemId = 0;
                    slots[i].amount = 0;
                }
                
                OnInventoryChanged?.Invoke();
                return true;
            }
        }
        
        // Item não encontrado
        return false;
    }
    
    public int GetItemCount(int itemId)
    {
        int count = 0;
        
        foreach (var slot in slots)
        {
            if (slot.itemId == itemId)
            {
                count += slot.amount;
            }
        }
        
        return count;
    }
    
    public List<InventorySlot> GetInventorySlots()
    {
        return slots;
    }
}
// GroundItem - Representa um item no chão
using UnityEngine;

public class GroundItem : MonoBehaviour
{
    [SerializeField] private int itemId;
    [SerializeField] private string itemName;
    [SerializeField] private int amount = 1;
    
    public void PickUp()
    {
        // Adiciona o item ao inventário do jogador
        bool success = InventoryManager.Instance.AddItem(itemId, amount);
        
        if (success)
        {
            // Notifica o jogador
            EventManager.Instance.TriggerEvent("ItemPickedUp", new ItemPickupInfo(itemId, itemName, amount));
            
            // Destroi o objeto
            Destroy(gameObject);
        }
        else
        {
            Debug.Log("Inventário cheio ou item inválido");
            // Você pode implementar uma mensagem para o jogador aqui
        }
    }
}

public class ItemPickupInfo
{
    public int ItemId { get; private set; }
    public string ItemName { get; private set; }
    public int Amount { get; private set; }
    
    public ItemPickupInfo(int itemId, string itemName, int amount)
    {
        ItemId = itemId;
        ItemName = itemName;
        Amount = amount;
    }
}
// GameManager - Controla o fluxo principal do jogo
using UnityEngine;

public class GameManager : MonoBehaviour
{
    public static GameManager Instance { get; private set; }
    
    [SerializeField] private PlayerManager playerManager;
    [SerializeField] private MonsterManager monsterManager;
    [SerializeField] private CombatManager combatManager;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public void Initialize()
    {
        // Inicializa todos os subsistemas na ordem correta
        playerManager.Initialize();
        monsterManager.Initialize();
        combatManager.Initialize();
    }
}
// GameInitializer - Inicializa todos os sistemas do jogo
using UnityEngine;

public class GameInitializer : MonoBehaviour
{
    [SerializeField] private GameManager gameManager;
    [SerializeField] private EventManager eventManager;
    [SerializeField] private UIManager uiManager;
    [SerializeField] private InventoryManager inventoryManager;
    [SerializeField] private ItemDatabase itemDatabase;
    
    private void Start()
    {
        // Verifica se todos os componentes necessários estão presentes
        if (gameManager == null || eventManager == null || uiManager == null ||
            inventoryManager == null || itemDatabase == null)
        {
            Debug.LogError("Algum componente está faltando. Verifique as referências.");
            return;
        }
        
        // Inicializa o jogo
        gameManager.Initialize();
        
        Debug.Log("Jogo inicializado com sucesso!");
    }
}
// EventManager - Sistema central de comunicação entre componentes
using System;
using System.Collections.Generic;
using UnityEngine;

public class EventManager : MonoBehaviour
{
    public static EventManager Instance { get; private set; }
    
    // Dicionário de eventos
    private Dictionary<string, Action<object>> eventDictionary = new Dictionary<string, Action<object>>();
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    // Registrar um listener para um evento
    public void AddListener(string eventName, Action<object> listener)
    {
        if (eventDictionary.ContainsKey(eventName))
        {
            eventDictionary[eventName] += listener;
        }
        else
        {
            eventDictionary.Add(eventName, listener);
        }
    }
    
    // Remover um listener
    public void RemoveListener(string eventName, Action<object> listener)
    {
        if (eventDictionary.ContainsKey(eventName))
        {
            eventDictionary[eventName] -= listener;
        }
    }
    
    // Disparar um evento
    public void TriggerEvent(string eventName, object data = null)
    {
        if (eventDictionary.ContainsKey(eventName))
        {
            eventDictionary[eventName]?.Invoke(data);
        }
    }
}
// CombatManager - Gerencia o sistema de combate
using UnityEngine;

public class CombatManager : MonoBehaviour
{
    public static CombatManager Instance { get; private set; }
    
    [SerializeField] private float attackRange = 2f;
    [SerializeField] private LayerMask targetLayers;
    
    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }
    
    public void Initialize()
    {
        // Inicialização do sistema de combate
        Debug.Log("Combat system initialized");
    }
    
    public bool IsInAttackRange(Transform attacker, Transform target)
    {
        return Vector3.Distance(attacker.position, target.position) <= attackRange;
    }
    
    public int CalculateDamage(CharacterStats attacker, CharacterStats defender, bool isMagic = false)
    {
        DerivedStats attackerStats = attacker.GetDerivedStats();
        DerivedStats defenderStats = defender.GetDerivedStats();
        
        int baseDamage = isMagic ? attackerStats.MATK : attackerStats.ATK;
        int defense = isMagic ? defenderStats.MDEF : defenderStats.DEF;
        
        // Cálculo básico de dano baseado no rAthena
        float hitChance = attackerStats.HIT / defenderStats.FLEE * 100f;
        
        // Verifica se o ataque acerta
        if (Random.Range(0f, 100f) > hitChance)
        {
            return 0; // Miss
        }
        
        // Fórmula de dano
        int damage = Mathf.Max(baseDamage - defense, 1);
        
        // Verifica crítico
        if (Random.Range(0f, 100f) < attackerStats.CRIT)
        {
            damage = (int)(damage * 1.5f);
            EventManager.Instance.TriggerEvent("CriticalHit", attacker.gameObject);
        }
        
        // Adicione mais modificadores conforme necessário (equipamentos, buffs, etc)
        
        return damage;
    }
}
// CharacterStats - Base para player e monsters
using UnityEngine;
using System;

[Serializable]
public class BaseStats
{
    public int STR; // Força - aumenta ataque físico
    public int AGI; // Agilidade - aumenta velocidade de ataque e esquiva
    public int VIT; // Vitalidade - aumenta HP e defesa física
    public int INT; // Inteligência - aumenta ataque mágico e SP
    public int DEX; // Destreza - aumenta precisão e dano crítico
    public int LUK; // Sorte - aumenta chance de crítico
}

[Serializable]
public class DerivedStats
{
    public int MaxHP;      // Vida máxima
    public int CurrentHP;  // Vida atual
    public int MaxSP;      // SP máximo (mana)
    public int CurrentSP;  // SP atual
    public int ATK;        // Ataque físico
    public int MATK;       // Ataque mágico
    public int DEF;        // Defesa física
    public int MDEF;       // Defesa mágica
    public float ASPD;     // Velocidade de ataque
    public float HIT;      // Precisão
    public float FLEE;     // Esquiva
    public float CRIT;     // Taxa de crítico
}

public class CharacterStats : MonoBehaviour
{
    [SerializeField] protected BaseStats baseStats;
    [SerializeField] protected DerivedStats derivedStats;
    
    [SerializeField] protected int level = 1;
    [SerializeField] protected int experiencePoints = 0;
    [SerializeField] protected int experienceToNextLevel = 100;
    
    public event Action<int, int> OnHPChanged; // (current, max)
    public event Action<int, int> OnSPChanged; // (current, max)
    public event Action<int> OnLevelUp; // (newLevel)
    
    protected virtual void Start()
    {
        CalculateDerivedStats();
        
        // Inicializa HP e SP com valores máximos
        derivedStats.CurrentHP = derivedStats.MaxHP;
        derivedStats.CurrentSP = derivedStats.MaxSP;
    }
    
    public virtual void CalculateDerivedStats()
    {
        // Cálculos baseados no rAthena
        derivedStats.MaxHP = 40 + level * 5 + (baseStats.VIT * 3);
        derivedStats.MaxSP = 10 + level * 2 + (baseStats.INT * 2);
        
        derivedStats.ATK = baseStats.STR + (baseStats.DEX / 5) + (level / 4);
        derivedStats.MATK = baseStats.INT + (baseStats.INT / 2) + (level / 4);
        
        derivedStats.DEF = baseStats.VIT / 2 + (baseStats.AGI / 5);
        derivedStats.MDEF = baseStats.INT / 2 + (baseStats.VIT / 5);
        
        derivedStats.ASPD = 200 + (baseStats.AGI * 2);
        derivedStats.HIT = 175 + baseStats.DEX + (level / 2);
        derivedStats.FLEE = 100 + baseStats.AGI + (level / 4);
        derivedStats.CRIT = 3 + (baseStats.LUK / 3);
        
        // Notifica os observadores sobre a mudança
        OnHPChanged?.Invoke(derivedStats.CurrentHP, derivedStats.MaxHP);
        OnSPChanged?.Invoke(derivedStats.CurrentSP, derivedStats.MaxSP);
    }
    
    public void ModifyHP(int amount)
    {
        derivedStats.CurrentHP = Mathf.Clamp(derivedStats.CurrentHP + amount, 0, derivedStats.MaxHP);
        OnHPChanged?.Invoke(derivedStats.CurrentHP, derivedStats.MaxHP);
        
        if (derivedStats.CurrentHP <= 0)
        {
            Die();
        }
    }
    
    public void ModifySP(int amount)
    {
        derivedStats.CurrentSP = Mathf.Clamp(derivedStats.CurrentSP + amount, 0, derivedStats.MaxSP);
        OnSPChanged?.Invoke(derivedStats.CurrentSP, derivedStats.MaxSP);
    }
    
    public virtual void AddExperience(int amount)
    {
        experiencePoints += amount;
        
        // Verifica se subiu de nível
        while (experiencePoints >= experienceToNextLevel)
        {
            experiencePoints -= experienceToNextLevel;
            LevelUp();
        }
    }
    
    protected virtual void LevelUp()
    {
        level++;
        experienceToNextLevel = CalculateNextLevelExperience();
        
        // Recalcula os status derivados
        CalculateDerivedStats();
        
        // Notifica os observadores
        OnLevelUp?.Invoke(level);
    }
    
    protected virtual int CalculateNextLevelExperience()
    {
        // Fórmula similar ao Ragnarok
        return 100 + (level * level * 20);
    }
    
    protected virtual void Die()
    {
        // Implementação específica nas classes filhas
        Debug.Log($"{gameObject.name} morreu");
    }
    
    // Getters para os status (útil para UI e cálculos)
    public int GetLevel() => level;
    public BaseStats GetBaseStats() => baseStats;
    public DerivedStats GetDerivedStats() => derivedStats;
}
// CameraController - Gerencia o comportamento da câmera
using UnityEngine;

public class CameraController : MonoBehaviour
{
    [SerializeField] private Transform target;
    [SerializeField] private float distance = 10f;
    [SerializeField] private float minDistance = 5f;
    [SerializeField] private float maxDistance = 15f;
    [SerializeField] private float scrollSpeed = 1f;
    [SerializeField] private float rotationSpeed = 5f;
    [SerializeField] private float minVerticalAngle = 10f;
    [SerializeField] private float maxVerticalAngle = 80f;
    
    private float currentRotationX = 45f;
    private float currentRotationY = 0f;
    
    private void LateUpdate()
    {
        // Rotação da câmera com o botão direito do mouse
        if (Input.GetMouseButton(1))
        {
            currentRotationY += Input.GetAxis("Mouse X") * rotationSpeed;
            currentRotationX -= Input.GetAxis("Mouse Y") * rotationSpeed;
            
            // Limita o ângulo vertical
            currentRotationX = Mathf.Clamp(currentRotationX, minVerticalAngle, maxVerticalAngle);
        }
        
        // Zoom com o scroll do mouse
        float scroll = Input.GetAxis("Mouse ScrollWheel");
        distance -= scroll * scrollSpeed;
        distance = Mathf.Clamp(distance, minDistance, maxDistance);
        
        // Calcula a nova posição e rotação da câmera
        Quaternion rotation = Quaternion.Euler(currentRotationX, currentRotationY, 0);
        Vector3 position = target.position - rotation * Vector3.forward * distance;
        
        transform.position = position;
        transform.rotation = rotation;
    }
}
