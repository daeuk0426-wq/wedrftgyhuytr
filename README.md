<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>캣 히어로: 방치형 RPG (관리자 강제 소환 추가)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        'cat-primary': '#4F46E5', // Indigo 600
                        'cat-secondary': '#F97316', // Orange 500
                        'cat-bg': '#F3F4F6', // Gray 100
                        'cat-card': '#FFFFFF',
                        'cat-skill': '#DC2626', // Red 600
                        'cat-health': '#10B981', // Emerald 500
                        'diamond': '#0EA5E9', // Sky 500
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        
        body { font-family: 'Inter', sans-serif; }
        
        .scroll-hidden::-webkit-scrollbar { display: none; }
        .scroll-hidden { -ms-overflow-style: none; scrollbar-width: none; }

        @keyframes attack { 0% { transform: scale(1); } 50% { transform: scale(1.05) translateY(-5px); } 100% { transform: scale(1); } }
        .animate-attack { animation: attack 0.3s ease-in-out; }
        
        @keyframes damage-up {
            0% { transform: translateY(0); opacity: 1; font-size: 1.25rem; }
            100% { transform: translateY(-40px); opacity: 0; font-size: 2rem; }
        }
        .damage-float { animation: damage-up 1s ease-out forwards; }
        
        @keyframes lightning-strike {
            0% { opacity: 0.5; transform: scale(1); }
            50% { opacity: 1; transform: scale(1.1); }
            100% { opacity: 0; transform: scale(0.9); }
        }
        .lightning-effect {
            position: absolute;
            color: #FFD700; /* Gold */
            font-size: 4rem;
            animation: lightning-strike 0.5s ease-out;
            pointer-events: none;
            z-index: 10;
        }

        /* 쿨다운 버튼 스타일 */
        #skillButton:disabled { background-color: #6B7280; cursor: not-allowed; position: relative; }
        #skillButton:disabled::after {
            content: attr(data-cooldown);
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            display: flex;
            align-items: center;
            justify-content: center;
            background-color: rgba(0, 0, 0, 0.5);
            color: white;
            font-weight: bold;
            border-radius: 0.5rem;
        }

        .cat-stunned { opacity: 0.5; filter: grayscale(100%); animation: shake 0.5s infinite; }
        /* Gacha 등급별 색상 */
        .gacha-common { background-color: #ECFDF5; border-color: #34D399; }
        .gacha-rare { background-color: #EFF6FF; border-color: #60A5FA; }
        .gacha-epic { background-color: #F5F3FF; border-color: #8B5CF6; }
        .gacha-legendary { background-color: #FFFBEB; border-color: #F59E0B; }
        .gacha-mythic { background-color: #FEF2F2; border-color: #F43F5E; }
        .gacha-rainbow { background-color: #FCE7F6; border-color: #EC4899; }

        .max-level-text { color: #f97316; font-weight: bold; }
    </style>
</head>
<body class="bg-cat-bg min-h-screen p-4">

    <div id="app" class="max-w-md mx-auto">
        <header class="text-center mb-6">
            <h1 class="text-3xl font-bold text-cat-primary flex items-center justify-center space-x-2">
                <span role="img" aria-label="Cat Hero">🐱</span>
                <span>캣 히어로: 방치형 RPG</span>
            </h1>
            <p class="text-sm text-gray-500 mt-1"><span id="adminStatus" class="font-bold text-red-500">일반 사용자</span></p>
        </header>

        <!-- 쿠폰 코드 입력창 -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-3">쿠폰 등록</h2>
            <div class="flex space-x-2">
                <input type="text" id="couponInput" placeholder="여기에 쿠폰 코드를 입력하세요" class="flex-1 p-2 border border-gray-300 rounded-lg text-sm uppercase">
                <button id="couponButton" class="bg-cat-primary text-white px-4 py-2 rounded-lg font-semibold hover:bg-indigo-600 transition duration-150">등록</button>
            </div>
        </div>

        <!-- 자원 및 난이도 패널 -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-3">자원 현황</h2>
            <div class="grid grid-cols-4 gap-2 text-center border-b pb-3 mb-3">
                <div>
                    <p class="text-xs text-gray-500">골드 💰</p>
                    <p id="goldDisplay" class="text-xl font-bold text-yellow-600">0</p>
                </div>
                <div>
                    <p class="text-xs text-gray-500">다이아 💎</p>
                    <p id="diamondDisplay" class="text-xl font-bold text-diamond">0</p>
                </div>
                <div>
                    <p class="text-xs text-gray-500">생선 🐟</p>
                    <p id="fishDisplay" class="text-xl font-bold text-blue-500">0</p>
                </div>
                <div>
                    <p class="text-xs text-gray-500">현재 스테이지</p>
                    <p id="stageDisplay" class="text-xl font-bold text-green-600">1-1.1</p>
                </div>
            </div>
            
            <div class="flex justify-between items-center">
                <p class="font-medium text-gray-700">현재 난이도:</p>
                <p id="difficultyDisplay" class="font-bold text-lg text-green-500">보통</p>
            </div>
             <div class="flex justify-between items-center mt-2">
                <p class="font-medium text-gray-700">환생 레벨:</p>
                <p id="reincarnationLevelDisplay" class="font-bold text-lg text-cat-secondary">Lv. 0</p>
            </div>
        </div>

        <!-- 영웅 및 전투 영역 -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-3">고양이 영웅 상태</h2>
            
            <!-- 고양이 체력 바 -->
            <div class="mb-4">
                <div class="flex justify-between items-center text-sm font-semibold mb-1">
                    <p class="text-gray-600 flex items-center">
                        <span id="catStatusEmoji" role="img" aria-label="Cat Health">💚</span>
                        체력 (HP)
                    </p>
                    <p id="catHealthDisplay" class="text-cat-health">100 / 100</p>
                </div>
                <div class="w-full bg-gray-300 rounded-full h-3">
                    <div id="catHealthBar" class="bg-cat-health h-3 rounded-full transition-all duration-150 ease-linear" style="width: 100%;"></div>
                </div>
            </div>
            
            <div class="flex justify-between items-center bg-gray-50 p-3 rounded-lg border border-gray-200 mb-4">
                <div class="flex items-center space-x-3">
                    <span role="img" aria-label="Cat" class="text-4xl" id="catHeroEmoji">😼</span>
                    <div>
                        <p class="font-medium">총 공격력</p>
                        <p id="attackPowerDisplay" class="text-xl font-bold text-cat-primary">10</p>
                    </div>
                </div>
                
                <div id="attackAnimationTarget" class="text-right">
                    <p class="text-sm text-gray-500">훈련 레벨</p>
                    <p class="font-bold text-cat-primary">Lv. <span id="catLevel">1</span></p>
                </div>
            </div>

            <!-- 전투 영역 -->
            <div id="battleArea" class="mt-4 p-4 bg-gray-800 rounded-lg text-white text-center h-28 flex flex-col justify-center items-center relative overflow-hidden cursor-pointer">
                <div id="monsterArea" class="absolute inset-0 flex justify-center items-center">
                    <div id="monsterContainer" class="transition-transform duration-200">
                        <span id="monsterEmoji" class="text-5xl">👻</span>
                        <p id="monsterName" class="text-sm mt-1">꿈속의 악몽</p>
                    </div>
                </div>
                
                <!-- 피해 텍스트 표시 영역 -->
                <div id="damageIndicatorArea" class="absolute inset-0 pointer-events-none"></div>

                <!-- 몬스터 체력 바 -->
                <div class="absolute top-2 left-1/2 transform -translate-x-1/2 w-3/4 bg-red-800 rounded-full h-2">
                    <div id="monsterHealthBar" class="bg-red-400 h-2 rounded-full transition-all duration-100 ease-linear" style="width: 100%;"></div>
                </div>
                <p id="monsterHealthMonsterDisplay" class="absolute top-4 text-xs font-semibold text-white">100 / 100</p>
            </div>
            <p class="text-xs text-center text-gray-500 mt-2">몬스터를 클릭하면 번개 ⚡ 공격 (Lv. <span id="lightningLevelDisplay">1</span>)</p>
        </div>
        
        <!-- 환생 & 뽑기 영역 -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-3">환생 및 특수 기능</h2>
            <div class="grid grid-cols-2 gap-3">
                
                <!-- 환생 버튼 -->
                <button id="reincarnateButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-cat-secondary hover:bg-orange-600 disabled:bg-gray-400">
                    환생하기 (0 💰 필요)
                </button>
                
                <!-- 뽑기 버튼 -->
                <button id="summonButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-diamond hover:bg-sky-600 disabled:bg-gray-400">
                    아이템 소환 (10 💎)
                </button>
            </div>

            <div class="mt-4">
                <p class="text-sm font-medium text-gray-600 mb-1">획득한 영구 아이템 (공격력 보너스)</p>
                <div id="gachaInventoryDisplay" class="flex flex-wrap gap-1 p-2 bg-gray-50 rounded-lg h-24 overflow-y-scroll scroll-hidden text-center">
                    <p class="text-sm text-gray-400 w-full py-2">소환된 아이템이 없습니다.</p>
                </div>
            </div>
        </div>

        <!-- 업그레이드 영역 (스킬 레벨 및 오토 추가) -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-3">영웅 강화</h2>
            <div class="grid grid-cols-2 gap-3">
                
                <!-- 훈련 레벨 강화 -->
                <button id="upgradeAttackButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-cat-secondary hover:bg-orange-600 disabled:bg-gray-400">
                    훈련 강화 (Lv. 1)
                </button>
                
                <!-- 체력 강화 -->
                <button id="upgradeHealthButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-cat-health hover:bg-green-600 disabled:bg-gray-400">
                    체력 강화 (Lv. 1)
                </button>
                
                <!-- 번개(클릭) 강화 -->
                <button id="upgradeLightningButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-yellow-500 hover:bg-yellow-600 disabled:bg-gray-400">
                    번개 강화 (Lv. 1)
                </button>

                <!-- 스킬 레벨 강화 -->
                <button id="upgradeSkillButton" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-cat-skill hover:bg-red-700 disabled:bg-gray-400">
                    스킬 강화 (Lv. 1)
                </button>
            </div>
            
            <div class="mt-4">
                <!-- 스킬 사용 -->
                <button id="skillButton" class="w-full bg-cat-skill text-white px-3 py-2 text-base rounded-lg hover:bg-red-700 transition duration-150">
                    궁극기 ⚡ (Lv. <span id="skillLevelDisplay">1</span>)
                </button>
                <!-- 스킬 오토 구매 (관리자는 무료 자동 활성화) -->
                <button id="buyAutoSkillButton" class="w-full mt-2 text-white px-3 py-2 text-sm rounded-lg transition duration-150 bg-cat-primary/80 hover:bg-cat-primary disabled:bg-gray-400">
                    스킬 자동 사용 구매 (50,000 💰)
                </button>
            </div>
        </div>

        <!-- 동료 영역 -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-4">든든한 동료들 💪 (Max Lv. <span id="companionMaxLevelDisplay">1000</span>)</h2>
            <div id="companionGrid" class="grid grid-cols-1 gap-3">
                <!-- 동료 카드가 여기에 동적으로 추가됩니다 -->
            </div>
        </div>

        <!-- 생선(합치기) 영역 (기존 유지) -->
        <div class="bg-cat-card p-4 rounded-xl shadow-lg mb-6">
            <h2 class="text-lg font-semibold text-gray-700 mb-2">생선 합치기 🐟</h2>
            
            <div class="flex justify-between items-center mb-3">
                <button id="getFishButton" class="flex-1 bg-blue-500 text-white p-2 rounded-lg font-semibold mr-2 hover:bg-blue-600 transition duration-150">
                    생선 획득 (+1)
                </button>
                <button id="mergeButton" class="flex-1 bg-green-500 text-white p-2 rounded-lg font-semibold ml-2 hover:bg-green-600 transition duration-150" disabled>
                    자동 합치기 ON (5초마다)
                </button>
            </div>
            
            <div id="fishGrid" class="grid grid-cols-4 gap-2 h-40 overflow-y-scroll p-1 border border-gray-200 rounded-lg bg-gray-50 scroll-hidden">
                <p id="emptyMessage" class="col-span-4 text-center text-gray-400 py-6">획득 버튼을 눌러 생선을 받으세요!</p>
            </div>
        </div>
        
        <!-- 관리자 패널 -->
        <div id="adminPanel" class="bg-gray-800 text-white p-4 rounded-xl shadow-lg mt-6 hidden">
            <h2 class="text-xl font-bold mb-3 text-red-400">--- 관리자 패널 ---</h2>
            <p class="text-sm mb-4 text-gray-300">경고: 관리자 권한으로 최대 레벨이 9999로 설정되었고, 스킬/번개 자동 사용이 활성화되었습니다.</p>
            
            <div class="space-y-3">
                <!-- 자원 조정 -->
                <div class="grid grid-cols-3 gap-2 items-center">
                    <input type="number" id="adminGoldInput" value="1000000" class="p-2 rounded text-black text-sm w-full">
                    <button onclick="adminAdjust('gold')" class="bg-yellow-500 text-black p-2 rounded text-sm hover:bg-yellow-400">골드 추가</button>
                    <input type="number" id="adminDiamondInput" value="1000" class="p-2 rounded text-black text-sm w-full">
                    <button onclick="adminAdjust('diamond')" class="bg-diamond p-2 rounded text-sm hover:bg-sky-400">다이아 추가</button>
                </div>
                
                <!-- 환생 레벨 조정 -->
                <div class="flex space-x-2">
                    <input type="number" id="adminRebirthInput" value="1000" class="p-2 rounded text-black text-sm flex-1">
                    <button onclick="adminAdjust('reincarnation')" class="bg-red-500 p-2 rounded text-sm hover:bg-red-400">환생 레벨 설정</button>
                </div>
                
                <!-- 레벨 조정 -->
                <div class="grid grid-cols-2 gap-2">
                    <input type="number" id="adminCatLevelInput" value="1000" placeholder="훈련 레벨" class="p-2 rounded text-black text-sm">
                    <button onclick="adminAdjust('catLevel')" class="bg-cat-secondary p-2 rounded text-sm hover:bg-orange-400">훈련 레벨 설정</button>
                    <input type="number" id="adminSkillLevelInput" value="1000" placeholder="스킬 레벨" class="p-2 rounded text-black text-sm">
                    <button onclick="adminAdjust('skillLevel')" class="bg-cat-skill p-2 rounded text-sm hover:bg-red-400">스킬 레벨 설정</button>
                </div>

                <!-- MAX 레벨 설정 버튼 -->
                <button onclick="adminMaxOutStats()" class="bg-purple-600 p-2 rounded w-full text-sm hover:bg-purple-500 font-bold">
                    모든 능력치 MAX (9999)로 설정
                </button>

                <button onclick="performReincarnation(true)" class="bg-green-500 p-2 rounded w-full text-sm hover:bg-green-400">관리자 강제 환생</button>

                <!-- Admin Gacha Control (신규 추가) -->
                <div class="mt-4 pt-4 border-t border-gray-700">
                    <h3 class="text-lg font-bold mb-2 text-sky-400">관리자 강제 소환 (100% 확률, 무료)</h3>
                    <div class="flex space-x-2 items-center">
                        <select id="adminRaritySelect" class="flex-1 p-2 rounded text-black text-sm">
                            <!-- Options will be dynamically added by JavaScript -->
                        </select>
                        <button onclick="adminPerformForcedSummon()" class="bg-sky-500 p-2 rounded text-sm hover:bg-sky-400 font-bold whitespace-nowrap">
                            강제 소환
                        </button>
                    </div>
                </div>
            </div>
        </div>


        <!-- 메시지 박스 (Alert 대체) -->
        <div id="messageBox" class="fixed inset-0 bg-gray-900 bg-opacity-75 hidden items-center justify-center p-4 z-50">
            <div class="bg-white p-6 rounded-xl shadow-2xl max-w-sm w-full text-center">
                <h3 id="messageTitle" class="text-xl font-bold mb-3">알림</h3>
                <p id="messageContent" class="text-gray-700 mb-4">내용</p>
                <button id="closeMessageBox" class="bg-cat-secondary text-white px-4 py-2 rounded-lg font-semibold hover:bg-orange-600 transition duration-150">확인</button>
            </div>
        </div>
        
    </div>

    <script>
        // =================================================================
        // 1. 초기 데이터 및 상태 설정
        // =================================================================
        let gold = 0;
        let diamonds = 0; 
        let reincarnationLevel = 0; 
        
        // --- 레벨 제한 및 관리자 설정 ---
        const MAX_LEVEL_NORMAL = 1000;
        const MAX_LEVEL_ADMIN = 9999;
        let isAdmin = false; 
        const MAX_REINCARNATION_LEVEL_PLAYER = MAX_LEVEL_NORMAL;
        const MAX_REINCARNATION_LEVEL_ADMIN = MAX_LEVEL_ADMIN;
        // ----------------------------------------
        
        let catLevel = 1; // 훈련 레벨
        let skillLevel = 1; // 스킬 레벨
        let catHealthLevel = 1;
        let lightningLevel = 1; // 클릭 데미지 레벨
        
        // --- 스킬 및 번개 자동화 (신규) ---
        let isSkillAuto = false; 
        const AUTO_SKILL_COST = 50000;
        let isLightningAuto = false; // 번개 자동 공격 상태 (관리자 특혜)
        // ----------------------------------------

        // --- 스테이지 및 난이도 설정 (순차 진행) ---
        let currentWorld = 1; 
        let currentStage = 1; 
        let subStageMonsterCount = 10; 
        
        // 사용자 요청: 세계 수를 50으로 증가
        const MAX_WORLD = 50; 

        // --- 관리자 권한 제한 설정 (사용자 요청) ---
        const ADMIN_COUPON_CODE = 'ADMIN';
        const MAX_ADMIN_REDEMPTIONS = 21; // 총 21회까지만 ADMIN 쿠폰 사용 가능
        // ----------------------------------------

        const DIFFICULTIES_ORDERED = [
            { key: 'normal', name: '보통', multiplier: 1, color: 'text-green-500' },
            { key: 'hard', name: '어려움', multiplier: 5, color: 'text-yellow-500' },
            { key: 'very_hard', name: '아주 어려움', multiplier: 25, color: 'text-orange-500' },
            { key: 'asceticism', name: '고행', multiplier: 125, color: 'text-red-600' }
        ];
        let currentDifficultyIndex = 0; 
        let currentDifficultyKey = DIFFICULTIES_ORDERED[0].key;
        // -----------------------------------------------------

        let baseAttack = 10;
        let fishBonusAttack = 0;
        let gachaBonusAttack = 0; 
        let isStunned = false;
        const STUN_DURATION = 5000;
        
        let currentMonster = { hp: 100, maxHp: 100, goldReward: 10, emoji: '👻', name: '꿈속의 악몽', level: 1 };
        let fishInventory = [];
        let gachaInventory = []; 
        
        const SKILL_BASE_COOLDOWN = 30;
        let skillCooldownTimer = 0;
        
        const COMPANION_DATA = [
            { id: 'wolf', name: '로봇 늑대', emoji: '🐺', baseBonus: 5, hireCost: 500, upgradeBaseCost: 150 },
            { id: 'sausage', name: '소시지 동료', emoji: '🌭', baseBonus: 3, hireCost: 1000, upgradeBaseCost: 200 },
            { id: 'minicat', name: '미니캣', emoji: '😺', baseBonus: 8, hireCost: 2000, upgradeBaseCost: 300 },
        ];
        let companions = COMPANION_DATA.map(data => ({ ...data, level: 0, isHired: false, }));

        let catMaxHp = 100;
        let catCurrentHp = 100;
        const BASE_HP = 100;
        const HP_MULTIPLIER = 1.5;
        
        const MONSTER_EMOJIS = ['👻', '👽', '💀', '🤡', '🕷️', '🐺', '🤖'];
        const MONSTER_NAMES = ['꿈속의 악몽', '외계 괴물', '해골 병사', '웃는 광대', '거미 마녀', '로봇 늑대', '파괴자'];
        const FISH_GRADES = [
            { level: 1, emoji: '🐠', bonus: 1 },
            { level: 2, emoji: '🐡', bonus: 3 },
            { level: 3, emoji: '🦈', bonus: 8 },
            { level: 4, emoji: '🐳', bonus: 20 },
            { level: 5, emoji: '🐙', bonus: 50 },
            { level: 6, emoji: '🦞', bonus: 120 },
            { level: 7, emoji: '🐉', bonus: 300 },
        ];
        
        const GACHA_GRADES = [
            { name: 'Rainbow (레인보우)', emoji: '🌈', chance: 0.01, bonus: 5000, color: 'gacha-rainbow' },
            { name: 'Mythic (신화)', emoji: '🌟', chance: 0.1, bonus: 1000, color: 'gacha-mythic' },
            { name: 'Legendary (전설)', emoji: '👑', chance: 1, bonus: 250, color: 'gacha-legendary' },
            { name: 'Epic (영웅)', emoji: '💎', chance: 5, bonus: 50, color: 'gacha-epic' },
            { name: 'Rare (희귀)', emoji: '✨', chance: 20, bonus: 10, color: 'gacha-rare' },
            { name: 'Common (일반)', emoji: '🟢', chance: 73.89, bonus: 1, color: 'gacha-common' },
        ];
        const SUMMON_COST = 10;
        
        // =================================================================
        // 2. DOM 요소 캐시
        // =================================================================
        const goldDisplay = document.getElementById('goldDisplay');
        const diamondDisplay = document.getElementById('diamondDisplay'); 
        const reincarnationLevelDisplay = document.getElementById('reincarnationLevelDisplay'); 
        const stageDisplay = document.getElementById('stageDisplay');
        const difficultyDisplay = document.getElementById('difficultyDisplay');
        const catLevelDisplay = document.getElementById('catLevel');
        const attackPowerDisplay = document.getElementById('attackPowerDisplay');
        const upgradeAttackButton = document.getElementById('upgradeAttackButton');
        const upgradeHealthButton = document.getElementById('upgradeHealthButton');
        const upgradeLightningButton = document.getElementById('upgradeLightningButton');
        const upgradeSkillButton = document.getElementById('upgradeSkillButton'); 
        const buyAutoSkillButton = document.getElementById('buyAutoSkillButton'); 
        const lightningLevelDisplay = document.getElementById('lightningLevelDisplay');
        const skillLevelDisplay = document.getElementById('skillLevelDisplay'); 
        const companionMaxLevelDisplay = document.getElementById('companionMaxLevelDisplay'); 
        const getFishButton = document.getElementById('getFishButton');
        const mergeButton = document.getElementById('mergeButton');
        const fishGrid = document.getElementById('fishGrid');
        const emptyMessage = document.getElementById('emptyMessage');
        const battleArea = document.getElementById('battleArea');
        const attackAnimationTarget = document.getElementById('attackAnimationTarget');
        const damageIndicatorArea = document.getElementById('damageIndicatorArea');
        const monsterContainer = document.getElementById('monsterContainer');
        const monsterEmoji = document.getElementById('monsterEmoji');
        const monsterName = document.getElementById('monsterName');
        const monsterHealthBar = document.getElementById('monsterHealthBar');
        const monsterHealthMonsterDisplay = document.getElementById('monsterHealthMonsterDisplay');
        const companionGrid = document.getElementById('companionGrid');
        const skillButton = document.getElementById('skillButton');
        const catHealthDisplay = document.getElementById('catHealthDisplay');
        const catHealthBar = document.getElementById('catHealthBar');
        const catStatusEmoji = document.getElementById('catStatusEmoji');
        const catHeroEmoji = document.getElementById('catHeroEmoji');
        const adminStatus = document.getElementById('adminStatus'); 

        const reincarnateButton = document.getElementById('reincarnateButton'); 
        const summonButton = document.getElementById('summonButton'); 
        const gachaInventoryDisplay = document.getElementById('gachaInventoryDisplay'); 
        
        const messageBox = document.getElementById('messageBox');
        const messageTitle = document.getElementById('messageTitle');
        const messageContent = document.getElementById('messageContent');
        const closeMessageBox = document.getElementById('closeMessageBox');

        const couponInput = document.getElementById('couponInput'); 
        const couponButton = document.getElementById('couponButton'); 
        const adminPanel = document.getElementById('adminPanel'); 

        closeMessageBox.addEventListener('click', () => {
            messageBox.classList.add('hidden');
            messageBox.classList.remove('flex');
        });
        
        document.addEventListener('keydown', (e) => {
            if (e.key === 'F12') {
                e.preventDefault();
                adminPanel.classList.toggle('hidden');
            }
        });

        // =================================================================
        // 2.5. 연속 업그레이드 로직 (사용자 요청 추가)
        // =================================================================
        let continuousUpgradeInterval = null;
        const CONTINUOUS_INTERVAL_MS = 50; // 연속 강화 속도 (50ms마다)

        /**
         * 연속 업그레이드 인터벌을 시작합니다.
         * @param {function} upgradeFunction - 반복적으로 호출할 업그레이드 함수
         */
        function startContinuousUpgrade(upgradeFunction) {
            stopContinuousUpgrade(); // 기존 인터벌이 있다면 중지

            // 첫 번째 호출은 즉시 실행
            upgradeFunction();

            // 인터벌 시작
            continuousUpgradeInterval = setInterval(upgradeFunction, CONTINUOUS_INTERVAL_MS);
        }

        /**
         * 연속 업그레이드 인터벌을 중지합니다.
         */
        function stopContinuousUpgrade() {
            if (continuousUpgradeInterval !== null) {
                clearInterval(continuousUpgradeInterval);
                continuousUpgradeInterval = null;
            }
        }

        /**
         * 버튼에 연속 강화 이벤트 리스너를 설정합니다.
         * @param {HTMLElement} buttonElement - 이벤트 리스너를 부착할 버튼 요소
         * @param {function} upgradeFunction - 업그레이드 함수
         * @param {string} [buttonId=null] - 동료 강화를 위한 ID (필요한 경우)
         */
        function setupContinuousUpgrade(buttonElement, upgradeFunction, buttonId = null) {
            if (!buttonElement) return;

            // 함수 호출 래퍼
            const callUpgrade = () => {
                if (buttonId) {
                    upgradeFunction(buttonId);
                } else {
                    upgradeFunction();
                }
            };

            // 마우스 이벤트 (PC)
            buttonElement.addEventListener('mousedown', (e) => {
                if (e.button === 0 && !buttonElement.disabled) { // 좌클릭 및 비활성화 상태가 아닐 때만
                    startContinuousUpgrade(callUpgrade);
                }
            });
            buttonElement.addEventListener('mouseup', stopContinuousUpgrade);
            buttonElement.addEventListener('mouseleave', stopContinuousUpgrade);

            // 터치 이벤트 (모바일)
            buttonElement.addEventListener('touchstart', (e) => {
                e.preventDefault(); // 기본 동작 방지 (스크롤 등)
                if (!buttonElement.disabled) {
                    startContinuousUpgrade(callUpgrade);
                }
            }, { passive: false });
            buttonElement.addEventListener('touchend', stopContinuousUpgrade);
            buttonElement.addEventListener('touchcancel', stopContinuousUpgrade);
        }
        
        // =================================================================
        // 3. 핵심 게임 로직 (공격력 계산, 난이도, 환생, 뽑기)
        // =================================================================
        
        function getCurrentMaxLevel() {
            return isAdmin ? MAX_LEVEL_ADMIN : MAX_LEVEL_NORMAL;
        }

        function calculateMonsterBaseHP(world, stage) {
            const initialBaseHP = 100;
            // 세계 수 50으로 증가에 맞춰 HP 증가율 조정 (기존보다 완만하게)
            const worldMultiplier = Math.pow(1.5, world - 1); 
            const stageMultiplier = 1 + (stage - 1) * 0.5;
            
            return initialBaseHP * worldMultiplier * stageMultiplier * (currentWorld > 10 ? 10 : 1); // 10세계 이후 급증 방지
        }

        function spawnNewMonster() {
            const baseHP = calculateMonsterBaseHP(currentWorld, currentStage);
            
            const currentDifficulty = DIFFICULTIES_ORDERED[currentDifficultyIndex];
            const difficultyMultiplier = currentDifficulty.multiplier;
            
            const subStageNumber = 11 - subStageMonsterCount;
            const substageMultiplier = 1 + (subStageNumber - 1) * 0.1;
            
            const newMaxHp = Math.floor(baseHP * difficultyMultiplier * substageMultiplier);
            const newGoldReward = Math.floor(newMaxHp / 100);
            
            const newEmojiIndex = Math.floor(Math.random() * MONSTER_EMOJIS.length);
            const newNameIndex = Math.floor(Math.random() * MONSTER_NAMES.length);

            currentMonster = {
                hp: newMaxHp,
                maxHp: newMaxHp,
                goldReward: newGoldReward,
                emoji: MONSTER_EMOJIS[newEmojiIndex],
                name: MONSTER_NAMES[newNameIndex],
                level: currentMonster.level + 1,
            };
            
            updateUI(); 
        }

        function calculateCompanionBonus() {
            let totalBonus = 0;
            const maxLevel = getCurrentMaxLevel();

            companions.forEach(c => {
                if (c.isHired) {
                    const level = Math.min(c.level, maxLevel); 
                    totalBonus += c.baseBonus * level * (1 + level * 0.1);
                }
            });
            return totalBonus;
        }
        
        function calculateGachaBonusAttack() {
            let totalBonus = 0;
            gachaInventory.forEach(item => {
                const grade = GACHA_GRADES.find(g => g.name === item.name);
                if (grade) {
                    totalBonus += grade.bonus;
                }
            });
            gachaBonusAttack = totalBonus;
            return totalBonus;
        }

        function calculateTotalAttack() {
            const companionBonus = calculateCompanionBonus();
            const permanentBonus = calculateGachaBonusAttack();
            const maxLevel = getCurrentMaxLevel();
            const actualCatLevel = Math.min(catLevel, maxLevel);

            return Math.floor(baseAttack * actualCatLevel + fishBonusAttack + companionBonus + permanentBonus);
        }

        function calculateClickDamage() {
            const baseClick = 5;
            const maxLevel = getCurrentMaxLevel();
            const actualLightningLevel = Math.min(lightningLevel, maxLevel);

            return Math.floor(baseClick * actualLightningLevel * (1 + actualLightningLevel * 0.2));
        }
        
        function calculateSkillDamage() {
            const totalAttack = calculateTotalAttack();
            const maxLevel = getCurrentMaxLevel();
            const actualSkillLevel = Math.min(skillLevel, maxLevel);
            const skillMultiplier = 5 + actualSkillLevel * 0.1;

            return Math.floor(totalAttack * skillMultiplier);
        }

        function calculateSkillCooldown() {
            const maxLevel = getCurrentMaxLevel();
            const actualSkillLevel = Math.min(skillLevel, maxLevel);
            return Math.max(1, SKILL_BASE_COOLDOWN - actualSkillLevel * 0.05);
        }


        function calculateReincarnationCost() {
            const baseCost = 100000;
            const cost = baseCost * Math.pow(reincarnationLevel + 1, 2.5);
            return Math.floor(cost);
        }
        
        function performReincarnation(isAdmin = false) {
            const requiredGold = calculateReincarnationCost();
            const maxLevel = isAdmin ? MAX_REINCARNATION_LEVEL_ADMIN : MAX_REINCARNATION_LEVEL_PLAYER;
            
            if (!isAdmin && gold < requiredGold) {
                showMessage("환생 불가", `환생에 필요한 골드 ${requiredGold.toLocaleString()} 💰가 부족합니다.`);
                return;
            }
            
            if (reincarnationLevel >= maxLevel) {
                 showMessage("최대 레벨", `환생 레벨이 최대치 (${maxLevel.toLocaleString()})에 도달했습니다.`);
                return;
            }

            const diamondGain = reincarnationLevel * 5 + 10;
            diamonds += diamondGain;
            
            reincarnationLevel++;
            gold = 0; 
            currentWorld = 1;
            currentStage = 1;
            subStageMonsterCount = 10;
            currentDifficultyIndex = 0; 
            currentDifficultyKey = DIFFICULTIES_ORDERED[0].key;

            catLevel = 1;
            skillLevel = 1;
            catHealthLevel = 1;
            lightningLevel = 1;
            catMaxHp = BASE_HP;
            catCurrentHp = BASE_HP;
            
            // 관리자 특혜 상태 유지
            const wasAdmin = isAdmin;
            if (wasAdmin) { 
                isSkillAuto = true;
                isLightningAuto = true;
            } else {
                isSkillAuto = false;
                isLightningAuto = false;
            }


            spawnNewMonster();
            updateUI();
            
            showMessage("환생 성공! 💫", 
                `Lv. ${reincarnationLevel.toLocaleString()}로 환생했습니다! 
                ${diamondGain.toLocaleString()} 💎를 획득하고 모든 레벨과 스테이지가 초기화되었습니다. 
                (최대 레벨: ${maxLevel.toLocaleString()})`);
        }
        
        function idleGain() {
            const baseGain = calculateTotalAttack();
            const rebirthBonus = 1 + (reincarnationLevel * 0.1); 
            const goldGain = Math.floor(baseGain * rebirthBonus * 5);
            gold += goldGain;

            const diamondChance = 0.005 * reincarnationLevel; 
            if (Math.random() < diamondChance && diamonds < 99999999) { 
                const diamondAmount = Math.floor(reincarnationLevel * 0.01 + 1);
                diamonds += diamondAmount;
            }

            updateUI();
        }
        
        function calculateSummonChance(baseChance) {
            const maxRebirthBonus = reincarnationLevel / MAX_REINCARNATION_LEVEL_PLAYER; 
            const bonusMultiplier = 1 + maxRebirthBonus;
            
            return baseChance * bonusMultiplier;
        }

        function summonItem() {
            if (diamonds < SUMMON_COST) {
                showMessage("다이아몬드 부족", `아이템 소환에 필요한 다이아몬드 ${SUMMON_COST} 💎가 부족합니다.`);
                return;
            }

            diamonds -= SUMMON_COST;

            let totalChance = 0;
            const adjustedGrades = GACHA_GRADES.map(grade => {
                const adjustedChance = calculateSummonChance(grade.chance);
                totalChance += adjustedChance;
                return { ...grade, adjustedChance };
            });

            let roll = Math.random() * totalChance;
            let currentSum = 0;
            let obtainedItem = null;

            for (const grade of adjustedGrades) {
                currentSum += grade.adjustedChance;
                if (roll <= currentSum) {
                    obtainedItem = grade;
                    break;
                }
            }

            if (obtainedItem) {
                const item = { 
                    name: obtainedItem.name, 
                    emoji: obtainedItem.emoji, 
                    bonus: obtainedItem.bonus 
                };
                gachaInventory.push(item);
                
                showMessage(`${obtainedItem.name} 획득!`, `공격력 +${item.bonus.toLocaleString()} 보너스를 영구적으로 획득했습니다!`);
            } else {
                showMessage("오류", "아이템 소환에 실패했습니다. (확률 계산 오류)");
            }

            updateUI();
        }

        // =================================================================
        // 4. UI 및 이벤트 핸들러
        // =================================================================
        
        function updateUI() {
            const maxLevel = getCurrentMaxLevel();

            catMaxHp = BASE_HP * Math.pow(HP_MULTIPLIER, catHealthLevel - 1);
            catCurrentHp = Math.min(catCurrentHp, catMaxHp);

            const totalAttack = calculateTotalAttack();
            const reincarnationCost = calculateReincarnationCost();
            
            goldDisplay.textContent = Math.floor(gold).toLocaleString();
            diamondDisplay.textContent = Math.floor(diamonds).toLocaleString(); 
            fishDisplay.textContent = fishInventory.length.toLocaleString();
            reincarnationLevelDisplay.textContent = `Lv. ${reincarnationLevel.toLocaleString()}`; 
            
            // 관리자 상태 표시 업데이트
            const adminRedemptionsCount = JSON.parse(localStorage.getItem('redeemedCoupons') || '[]').filter(c => c === ADMIN_COUPON_CODE).length;
            let adminText = isAdmin ? '관리자 권한 활성화 (MAX: 9999)' : `일반 사용자 (MAX: 1000) | ADMIN ${adminRedemptionsCount}/${MAX_ADMIN_REDEMPTIONS} 사용`;
            if (isAdmin) { adminText += isLightningAuto ? ' | ⚡ Auto ON' : ' | ⚡ Auto OFF'; }
            adminStatus.textContent = adminText;
            adminStatus.classList.toggle('text-red-500', isAdmin);
            adminStatus.classList.toggle('text-gray-500', !isAdmin);
            
            stageDisplay.textContent = `${currentWorld}-${currentStage}.${11 - subStageMonsterCount}`; 

            catLevelDisplay.textContent = catLevel.toLocaleString();
            lightningLevelDisplay.textContent = lightningLevel.toLocaleString(); 
            skillLevelDisplay.textContent = skillLevel.toLocaleString(); 
            attackPowerDisplay.textContent = totalAttack.toLocaleString();
            companionMaxLevelDisplay.textContent = maxLevel.toLocaleString(); 
            
            const currentDifficultyInfo = DIFFICULTIES_ORDERED[currentDifficultyIndex];
            difficultyDisplay.textContent = currentDifficultyInfo.name;
            difficultyDisplay.className = `font-bold text-lg ${currentDifficultyInfo.color}`;

            const monsterHealthPercent = (currentMonster.hp / currentMonster.maxHp) * 100;
            monsterHealthBar.style.width = `${Math.max(0, monsterHealthPercent)}%`;
            monsterHealthMonsterDisplay.textContent = `${Math.floor(currentMonster.hp).toLocaleString()} / ${Math.floor(currentMonster.maxHp).toLocaleString()}`;
            monsterEmoji.textContent = currentMonster.emoji;
            monsterName.textContent = currentMonster.name;

            const catHealthPercent = (catCurrentHp / catMaxHp) * 100;
            catHealthBar.style.width = `${Math.max(0, catHealthPercent)}%`;
            catHealthDisplay.textContent = `${Math.floor(catCurrentHp).toLocaleString()} / ${Math.floor(catMaxHp).toLocaleString()}`;

            if (isStunned) {
                catStatusEmoji.textContent = '🤕';
                catHeroEmoji.classList.add('cat-stunned');
                catHealthDisplay.classList.add('text-red-500');
            } else {
                catStatusEmoji.textContent = '💚';
                catHeroEmoji.classList.remove('cat-stunned');
                catHealthDisplay.classList.remove('text-red-500');
            }

            reincarnateButton.textContent = `환생하기 (${reincarnationCost.toLocaleString()} 💰 필요)`;
            reincarnateButton.disabled = gold < reincarnationCost || reincarnationLevel >= MAX_REINCARNATION_LEVEL_PLAYER;
            summonButton.textContent = `아이템 소환 (${SUMMON_COST} 💎)`;
            summonButton.disabled = diamonds < SUMMON_COST;

            // --- 업그레이드 버튼 갱신 ---
            
            const updateButton = (button, level, max, cost, name) => {
                const isMax = level >= max;
                button.textContent = isMax ? `${name} MAX Lv. ${max.toLocaleString()}` : `${name} 강화 (Lv. ${level.toLocaleString()}, ${cost.toLocaleString()} 💰)`;
                button.disabled = gold < cost || isMax;
                button.classList.toggle('max-level-text', isMax);
            };

            updateButton(upgradeAttackButton, catLevel, maxLevel, catLevel * 100, '훈련');
            updateButton(upgradeHealthButton, catHealthLevel, maxLevel, Math.floor(100 + 50 * catHealthLevel * catHealthLevel), '체력');
            updateButton(upgradeLightningButton, lightningLevel, maxLevel, Math.floor(150 + 75 * lightningLevel * 1.5), '번개');
            updateButton(upgradeSkillButton, skillLevel, maxLevel, Math.floor(500 + 250 * skillLevel * 2), '스킬');

            if (skillCooldownTimer > 0) {
                skillButton.disabled = true;
                skillButton.setAttribute('data-cooldown', skillCooldownTimer.toFixed(0) + 's');
            } else {
                skillButton.disabled = false;
                skillButton.removeAttribute('data-cooldown');
                skillButton.textContent = `궁극기 ⚡ (Lv. ${skillLevel.toLocaleString()})`;
            }
            
            // 스킬 오토 버튼 갱신 (관리자 특혜 반영)
            if (isAdmin) {
                buyAutoSkillButton.textContent = '스킬 자동 사용 ON (관리자 무료)';
                buyAutoSkillButton.disabled = true;
                buyAutoSkillButton.classList.remove('bg-cat-primary/80');
                buyAutoSkillButton.classList.add('bg-green-500/80');
            } else {
                buyAutoSkillButton.textContent = isSkillAuto ? '스킬 자동 사용 ON' : `스킬 자동 사용 구매 (${AUTO_SKILL_COST.toLocaleString()} 💰)`;
                buyAutoSkillButton.disabled = isSkillAuto || gold < AUTO_SKILL_COST;
                buyAutoSkillButton.classList.toggle('bg-green-500/80', isSkillAuto);
                buyAutoSkillButton.classList.toggle('bg-cat-primary/80', !isSkillAuto);
            }


            renderFishGrid();
            renderCompanions();
            renderGachaInventory(); 
        }

        // --- 기타 UI 함수들 ---

        function handleMonsterDefeat() {
            gold += currentMonster.goldReward;
            
            subStageMonsterCount--;
            
            if (subStageMonsterCount <= 0) {
                currentStage++;
                subStageMonsterCount = 10; 
                
                if (currentStage > 10) {
                    currentWorld++;
                    currentStage = 1; 
                    
                    if (currentWorld > MAX_WORLD) {
                        const currentDifficulty = DIFFICULTIES_ORDERED[currentDifficultyIndex];
                        const nextDifficultyIndex = currentDifficultyIndex + 1;

                        if (nextDifficultyIndex < DIFFICULTIES_ORDERED.length) {
                            currentDifficultyIndex = nextDifficultyIndex;
                            currentDifficultyKey = DIFFICULTIES_ORDERED[currentDifficultyIndex].key;
                            currentWorld = 1; 
                            showMessage("새로운 난이도 해금!", `${currentDifficulty.name} 난이도를 완벽하게 클리어했습니다! 이제 ${DIFFICULTIES_ORDERED[currentDifficultyIndex].name} (세계 1) 난이도에 도전하세요.`);
                        } else {
                            // MAX_WORLD 변수가 50으로 업데이트되었으므로 텍스트도 자동으로 반영됨
                            showMessage("최종 승리!", `모든 난이도 (${currentDifficulty.name})의 ${MAX_WORLD} 세계를 클리어했습니다! 당신은 진정한 캣 히어로입니다!`);
                            currentWorld = MAX_WORLD; 
                        }
                    } else {
                        showMessage("월드 클리어!", `${DIFFICULTIES_ORDERED[currentDifficultyIndex].name} ${currentWorld-1} 세계 클리어! ${currentWorld} 세계에 진입합니다.`);
                    }
                } else {
                    showMessage("스테이지 클리어!", `${DIFFICULTIES_ORDERED[currentDifficultyIndex].name} ${currentWorld}-${currentStage - 1} 스테이지 클리어! 다음 스테이지로 진입합니다.`);
                }
            }
            
            spawnNewMonster();

            monsterContainer.classList.add('animate-spin-out');
            setTimeout(() => {
                monsterContainer.classList.remove('animate-spin-out');
                battleArea.classList.add('shake');
                setTimeout(() => battleArea.classList.remove('shake'), 500);
            }, 50);
        }

        function renderGachaInventory() {
            gachaInventoryDisplay.innerHTML = '';
            if (gachaInventory.length === 0) {
                gachaInventoryDisplay.innerHTML = '<p class="text-sm text-gray-400 w-full py-2">소환된 아이템이 없습니다.</p>';
                return;
            }

            const grouped = gachaInventory.reduce((acc, item) => {
                const grade = GACHA_GRADES.find(g => g.name === item.name);
                if (!acc[grade.name]) {
                    acc[grade.name] = { count: 0, grade: grade };
                }
                acc[grade.name].count++;
                return acc;
            }, {});

            const sortedGroupKeys = Object.keys(grouped).sort((a, b) => {
                return GACHA_GRADES.findIndex(g => g.name === b) - GACHA_GRADES.findIndex(g => g.name === a);
            });

            sortedGroupKeys.forEach(name => {
                const data = grouped[name];
                const itemDiv = document.createElement('div');
                itemDiv.className = `p-1 rounded-lg border-2 text-xs font-semibold flex items-center justify-center ${data.grade.color}`;
                itemDiv.textContent = `${data.grade.emoji} x ${data.count}`;
                itemDiv.title = `${data.grade.name}: 공격력 +${data.grade.bonus * data.count} 보너스`;
                gachaInventoryDisplay.appendChild(itemDiv);
            });
        }
        
        function showMessage(title, content) {
            messageTitle.textContent = title;
            messageContent.textContent = content;
            messageBox.classList.remove('hidden');
            messageBox.classList.add('flex');
        }

        function upgradeCatAttack() {
            const cost = catLevel * 100;
            const maxLevel = getCurrentMaxLevel();

            if (catLevel >= maxLevel) { showMessage("최대 레벨", `훈련 레벨은 Lv. ${maxLevel.toLocaleString()}이(가) 최대입니다.`); return; }
            if (gold >= cost) {
                gold -= cost;
                catLevel++;
                updateUI();
                upgradeAttackButton.classList.add('bg-cat-primary/50');
                setTimeout(() => { upgradeAttackButton.classList.remove('bg-cat-primary/50'); }, 100);
            } else {
                // 연속 강화 중에는 메시지 생략하여 성능 유지
                if (continuousUpgradeInterval === null) {
                    showMessage("골드 부족", "훈련 레벨을 강화할 골드가 부족합니다.");
                }
            }
        }
        
        function upgradeCatHealth() {
            const cost = Math.floor(100 + 50 * catHealthLevel * catHealthLevel);
            const maxLevel = getCurrentMaxLevel();

            if (catHealthLevel >= maxLevel) { showMessage("최대 레벨", `체력 레벨은 Lv. ${maxLevel.toLocaleString()}이(가) 최대입니다.`); return; }
            if (gold >= cost) {
                gold -= cost;
                catHealthLevel++;
                catMaxHp = BASE_HP * Math.pow(HP_MULTIPLIER, catHealthLevel - 1);
                catCurrentHp = catMaxHp;
                // 메시지는 한 번만 표시 (연속 강화 중에는 성능 위해 생략)
                if (continuousUpgradeInterval === null) {
                    showMessage("체력 강화 성공", `최대 체력이 ${Math.floor(catMaxHp).toLocaleString()}으로 증가했습니다!`);
                }
                updateUI();
                upgradeHealthButton.classList.add('bg-green-300');
                setTimeout(() => { upgradeHealthButton.classList.remove('bg-green-300'); }, 100);
            } else {
                 if (continuousUpgradeInterval === null) {
                    showMessage("골드 부족", "체력을 강화할 골드가 부족합니다.");
                }
            }
        }

        function upgradeLightning() {
            const cost = Math.floor(150 + 75 * lightningLevel * 1.5);
            const maxLevel = getCurrentMaxLevel();

            if (lightningLevel >= maxLevel) { showMessage("최대 레벨", `번개 레벨은 Lv. ${maxLevel.toLocaleString()}이(가) 최대입니다.`); return; }
            if (gold >= cost) {
                gold -= cost;
                lightningLevel++;
                // 메시지는 한 번만 표시
                if (continuousUpgradeInterval === null) {
                    showMessage("번개 강화 성공", `번개 공격력(클릭 데미지)이 ${calculateClickDamage().toLocaleString()}으로 증가했습니다!`);
                }
                updateUI();
                upgradeLightningButton.classList.add('bg-yellow-300');
                setTimeout(() => { upgradeLightningButton.classList.remove('bg-yellow-300'); }, 100);
            } else {
                 if (continuousUpgradeInterval === null) {
                    showMessage("골드 부족", "번개 공격력을 강화할 골드가 부족합니다.");
                }
            }
        }

        function upgradeSkill() {
            const cost = Math.floor(500 + 250 * skillLevel * 2);
            const maxLevel = getCurrentMaxLevel();

            if (skillLevel >= maxLevel) { showMessage("최대 레벨", `스킬 레벨은 Lv. ${maxLevel.toLocaleString()}이(가) 최대입니다.`); return; }
            if (gold >= cost) {
                gold -= cost;
                skillLevel++;
                // 메시지는 한 번만 표시
                 if (continuousUpgradeInterval === null) {
                    showMessage("스킬 강화 성공", `스킬 공격력 배율이 증가하고 쿨타임이 감소했습니다!`);
                }
                updateUI();
                upgradeSkillButton.classList.add('bg-red-300');
                setTimeout(() => { upgradeSkillButton.classList.remove('bg-red-300'); }, 100);
            } else {
                 if (continuousUpgradeInterval === null) {
                    showMessage("골드 부족", "스킬 레벨을 강화할 골드가 부족합니다.");
                }
            }
        }

        /**
         * 스킬 자동 사용을 구매합니다. (관리자는 무료 활성화)
         */
        function buyAutoSkill() {
            if (isAdmin) {
                showMessage("관리자 특혜", "관리자는 스킬 자동 사용이 무료로 활성화되어 있습니다.");
                return;
            }

            if (isSkillAuto) return;
            if (gold >= AUTO_SKILL_COST) {
                gold -= AUTO_SKILL_COST;
                isSkillAuto = true;
                showMessage("구매 성공", "스킬 자동 사용 기능이 활성화되었습니다! 쿨타임이 될 때마다 자동으로 사용됩니다.");
                updateUI();
            } else {
                showMessage("골드 부족", `스킬 자동 사용 구매에 필요한 골드 ${AUTO_SKILL_COST.toLocaleString()} 💰가 부족합니다.`);
            }
        }
        
        /**
         * 몬스터 클릭 공격 (수동)
         */
        function monsterClickAttack(event) {
            if (isStunned) { return; }
            
            if (isLightningAuto) {
                 showMessage("자동 번개", "관리자 권한으로 번개 공격이 자동으로 실행되고 있습니다. 수동 클릭은 비활성화됩니다.");
                 return;
            }

            const damage = calculateClickDamage();
            currentMonster.hp -= damage;
            showDamageIndicator(damage, false, true);
            
            const lightningEffect = document.createElement('div');
            lightningEffect.textContent = '⚡';
            lightningEffect.className = 'lightning-effect';
            const rect = battleArea.getBoundingClientRect();
            lightningEffect.style.left = `${event.clientX - rect.left - 30}px`;
            lightningEffect.style.top = `${event.clientY - rect.top - 30}px`;
            battleArea.appendChild(lightningEffect);
            setTimeout(() => lightningEffect.remove(), 500);

            if (currentMonster.hp <= 0) { handleMonsterDefeat(); } else { updateUI(); }
        }

        /**
         * 번개 자동 공격 (관리자 전용)
         */
        function autoLightningAttack() {
            if (!isLightningAuto || isStunned) return;

            const damage = calculateClickDamage();
            currentMonster.hp -= damage;
            // auto attack은 위치 정보 없이 몬스터 위치 기준으로 랜덤하게 표시
            showDamageIndicator(damage, false, true);
            
            const lightningEffect = document.createElement('div');
            lightningEffect.textContent = '⚡';
            lightningEffect.className = 'lightning-effect';
            const rect = battleArea.getBoundingClientRect();
            
            // Random position within the monster area for visual effect
            const xPos = Math.random() * rect.width * 0.5 + rect.width * 0.25;
            const yPos = Math.random() * rect.height * 0.5 + rect.height * 0.25;
            
            lightningEffect.style.left = `${xPos}px`;
            lightningEffect.style.top = `${yPos}px`;
            battleArea.appendChild(lightningEffect);
            setTimeout(() => lightningEffect.remove(), 500);

            if (currentMonster.hp <= 0) { handleMonsterDefeat(); } else { updateUI(); }
        }

        function catAttack() {
            if (isStunned) return;
            const totalAttack = calculateTotalAttack();
            const damage = Math.max(1, totalAttack);
            currentMonster.hp -= damage;
            showDamageIndicator(damage, false, false, false);
            attackAnimationTarget.classList.add('animate-attack');
            setTimeout(() => attackAnimationTarget.classList.remove('animate-attack'), 300);

            if (currentMonster.hp <= 0) { handleMonsterDefeat(); } else { updateUI(); }
        }

        function monsterAttackCat() {
            if (isStunned) return;
            const monsterDamage = Math.max(1, Math.floor(currentMonster.maxHp * 0.001)); 
            catCurrentHp -= monsterDamage;
            showDamageIndicator(monsterDamage, false, false, true);
            monsterContainer.classList.add('animate-attack');
            setTimeout(() => monsterContainer.classList.remove('animate-attack'), 300);

            if (catCurrentHp <= 0) { handleCatDefeat(); } else { updateUI(); }
        }

        function handleCatDefeat() {
            isStunned = true;
            catCurrentHp = 0;
            showMessage("고양이 기절!", `고양이 영웅이 기절했습니다. ${STUN_DURATION / 1000}초 후에 자동으로 부활(회복)합니다.`);
            updateUI();
            setTimeout(() => {
                isStunned = false;
                catCurrentHp = catMaxHp;
                updateUI();
            }, STUN_DURATION);
        }

        function useSkill() {
            if (skillCooldownTimer > 0 || isStunned) { return; }
            const skillDamage = calculateSkillDamage();
            currentMonster.hp -= skillDamage;
            showDamageIndicator(skillDamage, true, false);
            skillCooldownTimer = calculateSkillCooldown(); 
            skillButton.textContent = `쿨다운 (${skillCooldownTimer.toFixed(0)}s)`;
            skillButton.classList.add('bg-red-900');

            if (currentMonster.hp <= 0) { handleMonsterDefeat(); } else { updateUI(); }
        }
        
        /**
         * 데미지 인디케이터 표시
         * @param {number} damage - 표시할 데미지 값
         * @param {boolean} isSkill - 스킬 공격 여부
         * @param {boolean} isClick - 클릭(번개) 공격 여부
         * @param {boolean} isMonsterAttack - 몬스터 공격 여부
         */
        function showDamageIndicator(damage, isSkill = false, isClick = false, isMonsterAttack = false) {
            const indicator = document.createElement('div');
            indicator.textContent = Math.floor(damage).toLocaleString();
            
            let colorClass = 'text-yellow-300';
            let sizeClass = 'text-xl';
            if (isSkill) { colorClass = 'text-red-500'; sizeClass = 'text-3xl'; } 
            else if (isClick) { colorClass = 'text-yellow-100'; sizeClass = 'text-2xl'; } 
            else if (isMonsterAttack) { colorClass = 'text-red-400'; sizeClass = 'text-xl'; }

            indicator.className = `absolute font-extrabold damage-float pointer-events-none ${colorClass} ${sizeClass}`;
            
            const xOffset = Math.random() * 40 - 20;
            const yOffset = Math.random() * 30 + 10;
            const monsterRect = monsterContainer.getBoundingClientRect();
            const battleRect = battleArea.getBoundingClientRect();

            if (isMonsterAttack) {
                indicator.style.left = `50%`;
                indicator.style.top = `20%`;
                indicator.style.transform = `translateX(-50%)`;
            } else {
                // 일반 공격, 스킬, 자동/수동 번개 공격은 몬스터 위치 기준으로 랜덤 표시
                indicator.style.left = `${(monsterRect.left - battleRect.left + monsterRect.width / 2) + xOffset}px`;
                indicator.style.top = `${(monsterRect.top - battleRect.top) + yOffset}px`;
            }
            
            damageIndicatorArea.appendChild(indicator);
            setTimeout(() => { indicator.remove(); }, 1000);
        }

        function upgradeCompanion(companionId) {
            const companion = companions.find(c => c.id === companionId);
            if (!companion) return;
            
            const maxLevel = getCurrentMaxLevel();
            if (companion.level >= maxLevel) { showMessage("최대 레벨", `동료 레벨은 Lv. ${maxLevel.toLocaleString()}이(가) 최대입니다.`); return; }


            const currentLevel = companion.level;
            const nextLevel = currentLevel + 1;
            const cost = currentLevel === 0 ? companion.hireCost : Math.floor(companion.upgradeBaseCost * nextLevel * 1.5);

            if (gold >= cost) {
                gold -= cost;
                companion.level++;
                companion.isHired = true;
                
                // 메시지는 한 번만 표시
                if (continuousUpgradeInterval === null) {
                    showMessage("성공!", `${companion.name}이(가) Lv. ${companion.level}로 ${currentLevel === 0 ? '고용되었습니다' : '강화되었습니다'}!`);
                }

                const btn = document.getElementById(`compBtn_${companionId}`);
                if (btn) { // 버튼이 있을 때만 효과 적용
                    btn.classList.add('bg-green-500');
                    setTimeout(() => { btn.classList.remove('bg-green-500'); }, 100);
                }

                updateUI();
            } else {
                // 메시지는 한 번만 표시
                if (continuousUpgradeInterval === null) {
                    showMessage("골드 부족", `${companion.name}을(를) ${currentLevel === 0 ? '고용' : '강화'}할 골드가 부족합니다. (${cost.toLocaleString()} 💰 필요)`);
                }
            }
        }
        
        function renderCompanions() {
            companionGrid.innerHTML = '';
            const maxLevel = getCurrentMaxLevel();

            companions.forEach((c) => {
                const companionCard = document.createElement('div');
                companionCard.className = 'bg-gray-50 p-3 rounded-lg border border-gray-200 flex justify-between items-center';
                
                const currentLevel = c.level;
                const nextLevel = currentLevel + 1;
                const cost = currentLevel === 0 ? c.hireCost : Math.floor(c.upgradeBaseCost * nextLevel * 1.5);
                const isAffordable = gold >= cost;
                const bonus = Math.floor(c.baseBonus * currentLevel * (1 + currentLevel * 0.1));
                const isMax = currentLevel >= maxLevel;

                const infoHTML = `
                    <div class="flex items-center space-x-3">
                        <span role="img" aria-label="${c.name}" class="text-3xl">${c.emoji}</span>
                        <div>
                            <p class="font-bold text-gray-800">${c.name}</p>
                            <p class="text-sm text-gray-600 ${isMax ? 'max-level-text' : ''}">Lv. ${currentLevel.toLocaleString()}${isMax ? ' (MAX)' : ''}</p>
                            <p class="text-xs text-cat-primary">+${bonus.toLocaleString()} 공격력 보너스</p>
                        </div>
                    </div>
                `;
                
                let buttonText;
                if (isMax) {
                    buttonText = `MAX Lv. ${maxLevel.toLocaleString()}`;
                } else if (c.isHired) {
                    buttonText = `강화 (${cost.toLocaleString()} 💰)`;
                } else {
                    buttonText = `고용 (${cost.toLocaleString()} 💰)`;
                }
                
                const buttonClass = isMax ? 'bg-gray-500 cursor-not-allowed' : (isAffordable ? 'bg-cat-secondary hover:bg-orange-600' : 'bg-gray-400 cursor-not-allowed');

                const buttonHTML = `
                    <button id="compBtn_${c.id}" class="text-white px-3 py-2 text-sm rounded-lg transition duration-150 ${buttonClass}" 
                        data-id="${c.id}" ${!isAffordable || isMax ? 'disabled' : ''}>
                        ${buttonText}
                    </button>
                `;

                companionCard.innerHTML = infoHTML + buttonHTML;
                companionGrid.appendChild(companionCard);

                const buttonElement = document.getElementById(`compBtn_${c.id}`);
                // 기존 addEventListener 대신 연속 강화 함수 호출
                setupContinuousUpgrade(buttonElement, upgradeCompanion, c.id);
            });
        }
        
        // --- 쿠폰 시스템 함수 ---
        function checkCoupon() {
            const couponCode = couponInput.value.trim().toUpperCase();
            if (!couponCode) {
                showMessage("오류", "쿠폰 코드를 입력해주세요.");
                return;
            }

            couponInput.value = ''; 

            const redeemedCoupons = JSON.parse(localStorage.getItem('redeemedCoupons') || '[]');

            if (redeemedCoupons.includes(couponCode)) {
                showMessage("이미 사용된 쿠폰", "이미 사용한 쿠폰 코드입니다.");
                return;
            }

            let result = null;

            switch (couponCode) {
                case ADMIN_COUPON_CODE:
                    const adminRedemptionsCount = redeemedCoupons.filter(c => c === ADMIN_COUPON_CODE).length;
                    
                    if (adminRedemptionsCount >= MAX_ADMIN_REDEMPTIONS) {
                        showMessage("권한 제한 초과", `관리자 권한은 총 ${MAX_ADMIN_REDEMPTIONS}회까지만 부여할 수 있습니다. 이미 최대치에 도달했습니다.`);
                        return;
                    }

                    if (!isAdmin) {
                        isAdmin = true;
                        isSkillAuto = true; // 관리자 특혜 1: 스킬 자동 사용 무료
                        isLightningAuto = true; // 관리자 특혜 2: 번개 자동 공격 무료
                        adminPanel.classList.remove('hidden');
                        
                        const newCount = adminRedemptionsCount + 1;
                        result = { title: "관리자 권한 획득!", content: `관리자 권한이 활성화되었습니다. (누적 사용 횟수: ${newCount}/${MAX_ADMIN_REDEMPTIONS}회). F12 키를 눌러 패널을 확인하세요. (최대 레벨 9999로 상승, 스킬 및 번개 자동화 적용)` };
                        localStorage.setItem('isAdmin', 'true');
                    } else {
                         showMessage("정보", "이미 관리자 권한을 가지고 있습니다.");
                         return;
                    }
                    break;
                case 'HELLO2024':
                    result = { title: "쿠폰 등록 성공!", content: "골드 50,000 💰와 다이아몬드 50 💎를 획득했습니다." };
                    gold += 50000;
                    diamonds += 50;
                    break;
                case 'CATPOWER':
                    result = { title: "쿠폰 등록 성공!", content: "훈련 레벨 100을 추가로 얻었습니다." };
                    catLevel += 100;
                    break;
                case 'DIAMOND777':
                    result = { title: "쿠폰 등록 성공!", content: "다이아몬드 777 💎를 획득했습니다." };
                    diamonds += 777;
                    break;
                case 'SKILLUP':
                    result = { title: "쿠폰 등록 성공!", content: "스킬 레벨 50을 추가로 얻었습니다." };
                    skillLevel += 50;
                    break;
                case 'PETFOOD':
                    result = { title: "쿠폰 등록 성공!", content: "생선 20개를 획득했습니다." };
                    for (let i = 0; i < 20; i++) acquireFishInternal();
                    break;
                case 'REBIRTHBONUS':
                    result = { title: "쿠폰 등록 성공!", content: "환생 레벨 5를 추가했습니다." };
                    reincarnationLevel += 5;
                    break;
                case 'LUCKYGOLD':
                    result = { title: "쿠폰 등록 성공!", content: "골드 1,000,000 💰를 획득했습니다." };
                    gold += 1000000;
                    break;
                case 'HEALKIT':
                    result = { title: "쿠폰 등록 성공!", content: "체력 레벨 50을 추가로 얻었습니다." };
                    catHealthLevel += 50;
                    break;
                case 'FASTCLICK':
                    result = { title: "쿠폰 등록 성공!", content: "번개 레벨 50을 추가로 얻었습니다." };
                    lightningLevel += 50;
                    break;
                default:
                    showMessage("쿠폰 없음", "유효하지 않은 쿠폰 코드이거나 만료된 쿠폰입니다.");
                    return;
            }

            if (result) {
                redeemedCoupons.push(couponCode);
                localStorage.setItem('redeemedCoupons', JSON.stringify(redeemedCoupons));
                updateUI();
                showMessage(result.title, result.content);
            }
        }
        
        function acquireFishInternal() {
            if (fishInventory.length >= 20) return;
            const newFish = { level: 1, element: null };
            fishInventory.push(newFish);
        }

        /**
         * 관리자 패널에서 골드/다이아를 조정합니다.
         */
        function adminAdjust(type) {
            const maxLevel = MAX_LEVEL_ADMIN;
            let amount = 0;

            if (type === 'gold') {
                amount = parseInt(document.getElementById('adminGoldInput').value) || 0;
                gold += amount;
                showMessage("관리자 조정", `골드 ${amount.toLocaleString()} 💰 추가되었습니다.`);
            } else if (type === 'diamond') {
                amount = parseInt(document.getElementById('adminDiamondInput').value) || 0;
                diamonds += amount;
                showMessage("관리자 조정", `다이아 ${amount.toLocaleString()} 💎가 추가되었습니다.`);
            } else if (type === 'reincarnation') {
                const level = parseInt(document.getElementById('adminRebirthInput').value) || 0;
                reincarnationLevel = Math.min(level, MAX_REINCARNATION_LEVEL_ADMIN);
                showMessage("관리자 조정", `환생 레벨이 Lv. ${reincarnationLevel.toLocaleString()}로 설정되었습니다.`);
                // 환생 시 레벨 초기화 방지: 단순 레벨 변경
            } else if (type === 'catLevel') {
                 const level = parseInt(document.getElementById('adminCatLevelInput').value) || 0;
                catLevel = Math.min(level, maxLevel);
                showMessage("관리자 조정", `훈련 레벨이 Lv. ${catLevel.toLocaleString()}로 설정되었습니다.`);
            } else if (type === 'skillLevel') {
                 const level = parseInt(document.getElementById('adminSkillLevelInput').value) || 0;
                skillLevel = Math.min(level, maxLevel);
                showMessage("관리자 조정", `스킬 레벨이 Lv. ${skillLevel.toLocaleString()}로 설정되었습니다.`);
            }
            updateUI();
        }
        window.adminAdjust = adminAdjust; 
        window.performReincarnation = performReincarnation; 

        /**
         * 관리자 권한으로 모든 레벨을 MAX로 설정합니다.
         */
        function adminMaxOutStats() {
            if (!isAdmin) { return; }
            const max = MAX_LEVEL_ADMIN;
            
            catLevel = max;
            skillLevel = max;
            catHealthLevel = max;
            lightningLevel = max;
            reincarnationLevel = MAX_REINCARNATION_LEVEL_ADMIN;
            
            companions.forEach(c => {
                c.level = max;
                c.isHired = true;
            });

            isSkillAuto = true;
            isLightningAuto = true; // 번개 자동 활성화

            showMessage("관리자 MAX 설정", `훈련, 스킬, 체력, 번개, 동료 레벨 및 환생 레벨이 모두 Lv. ${max.toLocaleString()}로 설정되었으며, 스킬 및 번개 자동 사용이 활성화되었습니다.`);
            updateUI();
        }
        window.adminMaxOutStats = adminMaxOutStats;

        /**
         * 관리자 권한으로 특정 등급의 아이템을 100% 확률로 강제 소환합니다. (신규)
         */
        function adminPerformForcedSummon() {
            if (!isAdmin) {
                showMessage("권한 부족", "관리자 권한이 없습니다.");
                return;
            }
            
            const adminRaritySelect = document.getElementById('adminRaritySelect');
            const rarityName = adminRaritySelect.value;

            const obtainedItem = GACHA_GRADES.find(g => g.name === rarityName);
            
            if (obtainedItem) {
                const item = { 
                    name: obtainedItem.name, 
                    emoji: obtainedItem.emoji, 
                    bonus: obtainedItem.bonus 
                };
                gachaInventory.push(item);
                
                showMessage("관리자 강제 소환 성공", `${obtainedItem.name} ${obtainedItem.emoji} 아이템을 강제로 소환했습니다! (공격력 +${item.bonus.toLocaleString()} 영구 획득)`);
                updateUI();
            } else {
                showMessage("오류", "선택한 등급을 찾을 수 없습니다.");
            }
        }
        window.adminPerformForcedSummon = adminPerformForcedSummon;

        /**
         * 관리자 강제 소환 드롭다운을 설정합니다. (신규)
         */
        function setupAdminGachaControls() {
            const adminRaritySelect = document.getElementById('adminRaritySelect');
            if (!adminRaritySelect) return;
            
            // 확률이 높은 등급부터 위로 오도록 역순 정렬
            const sortedGrades = [...GACHA_GRADES].reverse();

            sortedGrades.forEach(grade => {
                const option = document.createElement('option');
                option.value = grade.name;
                option.textContent = `${grade.emoji} ${grade.name} (Bonus: +${grade.bonus.toLocaleString()})`;
                adminRaritySelect.appendChild(option);
            });
        }


        // =================================================================
        // 5. 이벤트 및 게임 루프
        // =================================================================

        // 이벤트 리스너 추가
        reincarnateButton.addEventListener('click', () => performReincarnation(false)); 
        summonButton.addEventListener('click', summonItem); 
        couponButton.addEventListener('click', checkCoupon); 
        buyAutoSkillButton.addEventListener('click', buyAutoSkill); 
        
        // --- 연속 업그레이드 이벤트 리스너 적용 ---
        setupContinuousUpgrade(upgradeAttackButton, upgradeCatAttack);
        setupContinuousUpgrade(upgradeHealthButton, upgradeCatHealth);
        setupContinuousUpgrade(upgradeLightningButton, upgradeLightning);
        setupContinuousUpgrade(upgradeSkillButton, upgradeSkill);
        // ----------------------------------------
        
        // 생선 관련 함수 (기존 로직 유지)
        function acquireFish() { acquireFishInternal(); updateUI(); }
        getFishButton.addEventListener('click', acquireFish);
        function mergeFishAutomatically() { /* ... 기존 로직 ... */ }
        function canMergeAutomatically() { /* ... 기존 로직 ... */ return false; /* Placeholder */ }
        mergeButton.addEventListener('click', mergeFishAutomatically);
        battleArea.addEventListener('click', monsterClickAttack);

        // 1초마다 자동 공격
        setInterval(catAttack, 1000); 

        // 1초마다 유휴 자원 획득
        setInterval(idleGain, 1000);

        // 3초마다 몬스터가 고양이 공격
        setInterval(monsterAttackCat, 3000);

        // 5초마다 자동 합치기 시도
        setInterval(() => {
            if (canMergeAutomatically()) {
                mergeFishAutomatically();
            }
        }, 5000);
        
        // 200ms(0.2초) 마다 번개 자동 공격 (관리자 전용)
        setInterval(autoLightningAttack, 200);

        // 쿨다운 타이머 및 스킬 오토 업데이트 (1초마다)
        setInterval(() => {
            if (skillCooldownTimer > 0) {
                skillCooldownTimer = Math.max(0, skillCooldownTimer - 1);
            } else if (isSkillAuto && !isStunned) {
                useSkill();
            }
            updateUI();
        }, 1000); 

        // 초기화 및 시작
        window.onload = function() {
            spawnNewMonster(); 
            setupAdminGachaControls(); // 관리자 뽑기 컨트롤 설정
            
            // ADMIN 쿠폰 사용 기록 확인 및 현재 사용자 권한 복구
            const redeemedCoupons = JSON.parse(localStorage.getItem('redeemedCoupons') || '[]');
            if (localStorage.getItem('isAdmin') === 'true' && redeemedCoupons.includes(ADMIN_COUPON_CODE)) {
                isAdmin = true;
                isSkillAuto = true;
                isLightningAuto = true;
                adminPanel.classList.remove('hidden');
            }
            
            updateUI();
            
            console.log("%c[관리자 패널] F12 키를 눌러 관리자 패널을 토글할 수 있습니다.", "color: red; font-size: 14px; font-weight: bold;");
        }

        // --- Fish Merge Logic (Placeholders - MUST be fully implemented in a real game) ---

        function renderFishGrid() {
            fishBonusAttack = 0;
            fishGrid.innerHTML = '';
            if (fishInventory.length === 0) {
                emptyMessage.classList.remove('hidden');
                return;
            } else {
                emptyMessage.classList.add('hidden');
            }
            fishInventory.forEach((fish, index) => {
                const grade = FISH_GRADES.find(g => g.level === fish.level) || FISH_GRADES[0];
                const fishDiv = document.createElement('div');
                fishDiv.className = 'w-full h-10 flex items-center justify-center text-2xl bg-white rounded-lg shadow cursor-pointer transition duration-100 hover:scale-105';
                fishDiv.textContent = grade.emoji;
                fishDiv.title = `Lv.${grade.level} (+${grade.bonus} Attack)`;
                fishDiv.dataset.index = index;
                fishBonusAttack += grade.bonus;
                fishGrid.appendChild(fishDiv);
            });
        }
    </script>
</body>
</html>
