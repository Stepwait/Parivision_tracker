// Конфигурация
const CONFIG = {
  LIQUIPEDIA_API: "https://api.liquipedia.net/api/v3",
  LIQUIPEDIA_TOKEN: "api-key",
  TEAM_NAME: "PARIVISION",
  CACHE_DURATION: 24 * 60 * 60 * 1000, // 24 часа
  TIMEZONE: "Asia/Krasnoyarsk",
  WIKI: "counterstrike",
  LIMIT: 400,

  // ИНТЕРВАЛЫ ОБНОВЛЕНИЯ
  BACKGROUND_UPDATE_INTERVAL: 60 * 2000,        // 2 минуты
  USER_REQUEST_UPDATE_INTERVAL: 60 * 2000,      // для совместимости
  MIN_UPDATE_INTERVAL: 60 * 2000,                // 2 минуты

  MAX_RETRIES: 2,
  RETRY_DELAY: 5000,
  ALERT_EMAIL: "",
  SYNC_UPDATE_TIMEOUT: 15000,
  CACHE_AGE_FOR_UPDATE: 5 * 60 * 1000,
};

// Система логирования
const LOG_CONFIG = {
  MAX_LOGS: 50,
  LOG_LEVELS: {
    INFO: 'info',
    WARNING: 'warning',
    ERROR: 'error',
    SUCCESS: 'success',
    DEBUG: 'debug'
  }
};

// ================== ПЛАНИРОВЩИК ==================
function startBackgroundScheduler() {
  const cache = CacheService.getScriptCache();
  const hasTrigger = checkIfTriggerExists('runBackgroundUpdate');
  if (!hasTrigger) {
    cleanupTriggers();
    scheduleNextBackgroundUpdate();
    cache.put("scheduler_running", "true", 3600);
    logSystemEvent('Планировщик фоновых обновлений перезапущен', LOG_CONFIG.LOG_LEVELS.INFO);
  } else {
    cache.put("scheduler_running", "true", 3600);
  }
}

function checkIfTriggerExists(functionName) {
  const triggers = ScriptApp.getProjectTriggers();
  return triggers.some(trigger => trigger.getHandlerFunction() === functionName);
}

function cleanupTriggers() {
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    if (trigger.getHandlerFunction() === 'runBackgroundUpdate' || trigger.getHandlerFunction() === 'runForcedBackgroundUpdate') {
      ScriptApp.deleteTrigger(trigger);
    }
  });
}

function scheduleNextBackgroundUpdate() {
  const intervalMs = CONFIG.BACKGROUND_UPDATE_INTERVAL;
  console.log(`Scheduling next background update in ${intervalMs / 1000} seconds`);
  logSystemEvent(`Следующее фоновое обновление через ${intervalMs / 1000} сек`, LOG_CONFIG.LOG_LEVELS.INFO);
  try {
    deleteBackgroundUpdateTrigger('runBackgroundUpdate');
    ScriptApp.newTrigger('runBackgroundUpdate')
      .timeBased()
      .after(intervalMs)
      .create();
  } catch (error) {
    console.error('Failed to schedule next update:', error);
    logSystemEvent(`Ошибка планирования обновления: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
  }
}

// ================== ОСНОВНЫЕ ФУНКЦИИ ==================
function doGet(e) {
  startBackgroundScheduler();
  const params = e?.parameter || {};
  const functionName = params.function || params.func || 'getNextMatch';
  try {
    let result;
    switch (functionName) {
      case 'getNextMatch':
      case 'getAllUpcomingMatches':
        console.log('User request for', functionName);
        break;
      case 'getMatchesJSON':
        return ContentService.createTextOutput(JSON.stringify(getTeamMatches(false)))
          .setMimeType(ContentService.MimeType.JSON);
      case 'getSystemStatus':
      case 'updateStatus':
        result = JSON.stringify(getUpdateStatus());
        return ContentService.createTextOutput(result).setMimeType(ContentService.MimeType.JSON);
      default:
        console.log('Default request for', functionName);
    }
    switch (functionName) {
      case 'getNextMatch': result = getNextMatch(); break;
      case 'getAllUpcomingMatches': result = getAllUpcomingMatches(); break;
      case 'clearCache': result = clearCache(); break;
      case 'forceRefresh': result = forceRefresh(); break;
      case 'runBackgroundUpdate': result = runBackgroundUpdate(); break;
      case 'fixStaleCache': result = fixStaleCache(); break;
      case 'statusDashboard': return getStatusDashboard(params.refresh === 'true');
      case 'clearSystemLogs': result = clearSystemLogs(); break;
      case 'getStreamOverlay': return getStreamOverlay();
      default: result = getNextMatch();
    }
    return ContentService.createTextOutput(result).setMimeType(ContentService.MimeType.TEXT);
  } catch (error) {
    console.error('doGet Error:', error);
    logSystemEvent('doGet Error: ' + error.message, LOG_CONFIG.LOG_LEVELS.ERROR);
    return ContentService.createTextOutput(getCachedDataWithFallback()).setMimeType(ContentService.MimeType.TEXT);
  }
}

function doPost(e) { return doGet(e); }

function shouldSkipUpdate(updateType = 'background') {
  const cache = CacheService.getScriptCache();
  const updateInProgress = cache.get("update_in_progress");
  if (updateInProgress === "true") {
    console.log('Update skipped - already in progress');
    return true;
  }
  return false;
}

function runBackgroundUpdate() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();

  if (shouldSkipUpdate('background')) return "🔄 Обновление пропущено";

  const lock = LockService.getScriptLock();
  try {
    if (!lock.tryLock(10000)) {
      console.log('Background update skipped - lock not acquired');
      return "🔄 Обновление уже выполняется";
    }

    console.log('Background update started...');
    logSystemEvent('Запуск фонового обновления данных', LOG_CONFIG.LOG_LEVELS.INFO);

    cache.put("update_in_progress", "true", 120);
    checkAPILimit(); // проверка лимитов

    const allMatches = fetchAllMatchesWithRetry();

    // Если произошла ошибка и данные не получены – не обновляем кэш
    if (allMatches === null) {
      console.log('No data received from API, keeping old cache');
      logSystemEvent('Не удалось получить данные от API, сохранён старый кэш', LOG_CONFIG.LOG_LEVELS.WARNING);
      return "⚠️ Не удалось обновить данные, используется старый кэш";
    }

    const teamMatches = filterTeamMatches(allMatches);

    // Сохраняем новые данные (даже если массив пуст – это может быть правдой)
    const rawCacheData = {
      rawMatches: teamMatches,
      timestamp: now,
      source: 'background_update',
      matchesCount: teamMatches.length
    };

    cache.put("raw_matches_data", JSON.stringify(rawCacheData), CONFIG.CACHE_DURATION / 1000);
    cache.put("last_successful_update", now.toString(), 3600);
    cache.put("error_count", "0", 3600);

    console.log(`Background update completed. Matches: ${teamMatches.length}`);
    logSystemEvent(`Фоновое обновление завершено. Матчей: ${teamMatches.length}`, LOG_CONFIG.LOG_LEVELS.SUCCESS);
    return `✅ Данные обновлены. Матчей: ${teamMatches.length}`;

  } catch (error) {
    console.error('Background update failed:', error);
    logSystemEvent(`Ошибка фонового обновления: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return `❌ Ошибка: ${error.message}`;
  } finally {
    try { lock.releaseLock(); } catch (e) { }
    cache.remove("update_in_progress");
    deleteBackgroundUpdateTrigger('runBackgroundUpdate');
    scheduleNextBackgroundUpdate();
  }
}

// ================== ЛОГИРОВАНИЕ ==================
function logSystemEvent(message, level = LOG_CONFIG.LOG_LEVELS.INFO) {
  const cache = CacheService.getScriptCache();
  const timestamp = new Date().getTime();
  const logEntry = { timestamp, time: new Date().toLocaleString('ru-RU'), level, message, source: 'system' };
  let logs = [];
  const storedLogs = cache.get('system_logs');
  if (storedLogs) try { logs = JSON.parse(storedLogs); } catch (e) { }
  logs.unshift(logEntry);
  if (logs.length > LOG_CONFIG.MAX_LOGS) logs = logs.slice(0, LOG_CONFIG.MAX_LOGS);
  cache.put('system_logs', JSON.stringify(logs), 3600);
  console.log(`[${level.toUpperCase()}] ${message}`);
}

function getSystemLogs(limit = 20) {
  const cache = CacheService.getScriptCache();
  const storedLogs = cache.get('system_logs');
  if (!storedLogs) return [];
  try { return JSON.parse(storedLogs).slice(0, limit); } catch (e) { return []; }
}

function clearSystemLogs() {
  const cache = CacheService.getScriptCache();
  cache.remove('system_logs');
  logSystemEvent('Системные логи очищены', LOG_CONFIG.LOG_LEVELS.INFO);
  return "✅ Системные логи очищены";
}

// ================== API И КЭШИРОВАНИЕ ==================
function checkAPILimit() {
  const cache = CacheService.getScriptCache();
  const now = new Date();
  const today = now.toDateString();
  const dailyKey = `api_requests_${today}`;
  const hourlyKey = `api_requests_${now.getHours()}`;
  const dailyCount = parseInt(cache.get(dailyKey) || "0");
  const hourlyCount = parseInt(cache.get(hourlyKey) || "0");
  cache.put(dailyKey, (dailyCount + 1).toString(), 86400);
  cache.put(hourlyKey, (hourlyCount + 1).toString(), 3600);
  console.log(`API requests - Today: ${dailyCount + 1}, This hour: ${hourlyCount + 1}`);
  const DAILY_LIMIT = 2000;
  const HOURLY_LIMIT = 100;
  if (dailyCount >= DAILY_LIMIT) {
    const error = new Error(`Daily API limit reached: ${dailyCount}/${DAILY_LIMIT}`);
    logSystemEvent(error.message, LOG_CONFIG.LOG_LEVELS.ERROR);
    throw error;
  }
  if (hourlyCount >= HOURLY_LIMIT) {
    const error = new Error(`Hourly API limit reached: ${hourlyCount}/${HOURLY_LIMIT}`);
    logSystemEvent(error.message, LOG_CONFIG.LOG_LEVELS.ERROR);
    throw error;
  }
  return { daily: dailyCount + 1, hourly: hourlyCount + 1, dailyLimit: DAILY_LIMIT, hourlyLimit: HOURLY_LIMIT };
}

function getUpdateStatus() {
  const cache = CacheService.getScriptCache();
  const lastUpdate = cache.get("last_successful_update");
  const lastAttempt = cache.get("last_update_attempt");
  const lastForcedAttempt = cache.get("last_forced_attempt");
  const lastBackgroundAttempt = cache.get("last_background_attempt");
  const lastError = cache.get("last_update_error");
  const lastErrorDetails = cache.get("last_update_error_details");
  const lastErrorTime = cache.get("last_update_error_time");
  const errorCount = cache.get("error_count");
  const updateInProgress = cache.get("update_in_progress");
  const currentCache = cache.get("raw_matches_data");
  const now = new Date().getTime();
  const today = new Date().toDateString();
  const hourlyKey = `api_requests_${new Date().getHours()}`;
  const dailyCount = parseInt(cache.get(`api_requests_${today}`) || "0");
  const hourlyCount = parseInt(cache.get(hourlyKey) || "0");
  const usagePercent = Math.round((dailyCount / 800) * 100);
  let cacheInfo = null;
  if (currentCache) {
    try {
      const rawData = JSON.parse(currentCache);
      cacheInfo = {
        timestamp: new Date(rawData.timestamp).toLocaleString('ru-RU'),
        source: rawData.source,
        matchesCount: rawData.rawMatches ? rawData.rawMatches.length : 0,
        cacheAgeMinutes: Math.round((now - rawData.timestamp) / 60000)
      };
    } catch (e) { cacheInfo = { error: 'Cannot parse cache' }; }
  }
  return {
    lastSuccessfulUpdate: lastUpdate ? new Date(parseInt(lastUpdate)).toLocaleString('ru-RU') : 'никогда',
    lastUpdateAttempt: lastAttempt ? new Date(parseInt(lastAttempt)).toLocaleString('ru-RU') : 'никогда',
    lastForcedAttempt: lastForcedAttempt ? new Date(parseInt(lastForcedAttempt)).toLocaleString('ru-RU') : 'никогда',
    lastBackgroundAttempt: lastBackgroundAttempt ? new Date(parseInt(lastBackgroundAttempt)).toLocaleString('ru-RU') : 'никогда',
    lastError: lastError || 'нет',
    lastErrorDetails: lastErrorDetails || '',
    lastErrorTime: lastErrorTime ? new Date(parseInt(lastErrorTime)).toLocaleString('ru-RU') : 'N/A',
    errorCount: errorCount || '0',
    timeSinceLastUpdate: lastUpdate ? Math.round((now - parseInt(lastUpdate)) / 60000) + ' minutes' : 'N/A',
    backgroundUpdateEnabled: true,
    updateInProgress: updateInProgress === "true",
    cacheInfo: cacheInfo,
    apiUsage: { daily: dailyCount, hourly: hourlyCount, dailyLimit: 2000, hourlyLimit: 100, usagePercent: usagePercent },
    updateIntervals: {
      background: CONFIG.BACKGROUND_UPDATE_INTERVAL / 60000 + ' min',
      userRequest: CONFIG.USER_REQUEST_UPDATE_INTERVAL / 60000 + ' min',
      minUpdate: CONFIG.MIN_UPDATE_INTERVAL / 60000 + ' min'
    },
    systemLogs: getSystemLogs(15)
  };
}

function deleteBackgroundUpdateTrigger(functionName = 'runBackgroundUpdate') {
  try {
    const triggers = ScriptApp.getProjectTriggers();
    triggers.forEach(trigger => {
      if (trigger.getHandlerFunction() === functionName) ScriptApp.deleteTrigger(trigger);
    });
    console.log(`${functionName} trigger deleted`);
  } catch (error) { console.error('Error deleting trigger:', error); }
}

function getTeamMatches(shouldUpdate = false) {
  const cache = CacheService.getScriptCache();
  const rawCached = cache.get("raw_matches_data");
  if (shouldUpdate) {
    console.log('Forced update requested, fetching fresh data...');
    return fetchDataSynchronously();
  }
  if (rawCached) {
    try {
      const rawCacheData = JSON.parse(rawCached);
      const ageMinutes = Math.round((new Date().getTime() - rawCacheData.timestamp) / 60000);
      console.log('Using cached data, age:', ageMinutes + 'min');
      return processMatches(rawCacheData.rawMatches);
    } catch (e) {
      console.error('Error parsing raw cache, will try to refresh', e);
    }
  }
  console.log('No raw cache available, fetching data synchronously');
  return fetchDataSynchronously();
}

function fetchDataSynchronously() {
  try {
    console.log('Synchronous API fetch started');
    const allMatches = fetchAllMatchesWithRetry();
    if (allMatches === null) {
      throw new Error('No data received from API');
    }
    const teamMatches = filterTeamMatches(allMatches);
    const result = processMatches(teamMatches);
    const rawCacheData = {
      rawMatches: teamMatches,
      timestamp: new Date().getTime(),
      source: 'api_sync'
    };
    const cache = CacheService.getScriptCache();
    cache.put("raw_matches_data", JSON.stringify(rawCacheData), CONFIG.CACHE_DURATION / 1000);
    cache.put("last_successful_update", new Date().getTime().toString(), 3600);
    console.log('Synchronous fetch completed successfully');
    return result;
  } catch (error) {
    console.error('Synchronous fetch failed:', error);
    return { hasMatches: false, message: `Данные временно недоступны. Попробуйте обновить через несколько минут.`, matches: [], cacheUsed: false, apiError: true };
  }
}

function fetchAllMatchesWithDebug() {
  checkAPILimit();
  const now = new Date();
  const twoHoursAgo = new Date(now.getTime() - 2 * 60 * 60 * 1000);
  const oneMonthFromNow = new Date(now.getTime() + 30 * 24 * 60 * 60 * 1000);
  const formatDate = (date) => date.toISOString().split('T')[0];
  const dateFrom = formatDate(twoHoursAgo);
  const dateTo = formatDate(oneMonthFromNow);
  const conditions = encodeURIComponent(`[[date::>${dateFrom}]] AND [[date::<${dateTo}]]`);
  const fields = 'date,match2opponents,tournament,status,finished,winner,score,bestof,match2games';
  const url = `${CONFIG.LIQUIPEDIA_API}/match?wiki=${CONFIG.WIKI}&limit=${CONFIG.LIMIT}&conditions=${conditions}&fields=${fields}&order=date ASC`;
  console.log('Fetching from Liquipedia API:', url);
  const options = {
    method: 'GET',
    headers: {
      Authorization: `Apikey ${CONFIG.LIQUIPEDIA_TOKEN}`,
      'Accept-Encoding': 'gzip',
      'User-Agent': 'Team Matches Bot/2.0',
      Accept: 'application/json'
    },
    muteHttpExceptions: true,
    validateHttpsCertificates: false
  };
  try {
    const response = UrlFetchApp.fetch(url, options);
    const responseCode = response.getResponseCode();
    const contentText = response.getContentText();
    if (responseCode !== 200) throw new Error(`Liquipedia API ошибка: ${responseCode}`);
    const data = JSON.parse(contentText);
    if (!data || !data.result) throw new Error('Некорректный ответ от API');
    console.log('API returned', data.result.length, 'matches');
    return data.result;
  } catch (error) {
    console.error('API fetch error:', error);
    throw error;
  }
}

function fetchAllMatchesWithRetry() {
  for (let attempt = 1; attempt <= CONFIG.MAX_RETRIES; attempt++) {
    try {
      console.log(`API attempt ${attempt} of ${CONFIG.MAX_RETRIES}`);
      const result = fetchAllMatchesWithDebug();
      if (!result || !Array.isArray(result)) throw new Error('Invalid API response format');
      console.log(`API attempt ${attempt} successful, matches: ${result.length}`);
      return result;
    } catch (error) {
      console.error(`API attempt ${attempt} failed:`, error);
      if (attempt < CONFIG.MAX_RETRIES) {
        const delay = CONFIG.RETRY_DELAY * attempt;
        console.log(`Waiting ${delay}ms before retry...`);
        Utilities.sleep(delay);
      }
    }
  }
  console.error(`All ${CONFIG.MAX_RETRIES} API attempts failed, returning null`);
  logSystemEvent('Все попытки API не удались, возвращаем null', LOG_CONFIG.LOG_LEVELS.ERROR);
  return null; // ИЗМЕНЕНО: вместо пустого массива возвращаем null
}

function filterTeamMatches(matches) {
  if (!matches || !Array.isArray(matches)) return [];
  console.log('Filtering', matches.length, 'matches for team:', CONFIG.TEAM_NAME);
  const searchName = CONFIG.TEAM_NAME.toLowerCase().trim();
  return matches.filter(match => {
    if (!match || !match.match2opponents) return false;
    for (let opponent of match.match2opponents) {
      if (opponent && opponent.name) {
        const opponentName = opponent.name.toLowerCase().trim();
        if (opponentName === searchName) return true;
      }
    }
    return false;
  });
}

// ================== ОБРАБОТКА МАТЧЕЙ ==================
function processMatches(matches) {
  if (!matches || matches.length === 0) {
    return { hasMatches: false, message: `На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`, matches: [] };
  }
  const now = new Date();
  const formattedMatches = [];
  matches.forEach(match => {
    if (!match.match2opponents || match.match2opponents.length < 2) return;
    const teams = extractTeamsFromMatch(match);
    if (!teams) return;
    const matchDate = parseLiquipediaDate(match.date);
    const timeDiff = matchDate.getTime() - now.getTime();
    const hoursDiff = timeDiff / (1000 * 60 * 60);
    if (hoursDiff < -4) return;
    const matchStatus = getMatchStatus(match, hoursDiff);
    if (!matchStatus.isFinished) {
      let gamesInfo = null;
      let currentMap = null;
      let seriesScore = null;
      if (match.match2games && Array.isArray(match.match2games) && match.match2games.length > 0) {
        const games = match.match2games;
        if (match.score) seriesScore = match.score;

        // Карта считается завершённой, если есть победитель (winner не пустой)
        const finishedGames = games.filter(game => game.winner && game.winner !== '');
        // Текущая карта – первая, у которой нет победителя
        const liveGame = games.find(game => !game.winner || game.winner === '');

        // Функция для переупорядочивания счёта: первой всегда идёт наша команда
        const reorderScores = (gameScores) => {
          if (!gameScores || gameScores.length < 2) return gameScores;
          const parIndex = gameScores.findIndex(s => s.team === teams.team);
          if (parIndex === 0) return gameScores; // уже на первом месте
          return [gameScores[parIndex], gameScores[1 - parIndex]];
        };

        if (liveGame) {
          const rawScores = extractMapScores(liveGame, match.match2opponents);
          currentMap = {
            mapNumber: finishedGames.length + 1,
            totalMaps: match.bestof || 3,
            scores: reorderScores(rawScores)
          };
        }

        gamesInfo = games.map((game, index) => {
          const rawScores = extractMapScores(game, match.match2opponents);
          return {
            mapNumber: index + 1,
            finished: !!(game.winner && game.winner !== ''), // true если есть победитель
            winner: game.winner,
            duration: game.duration,
            scores: reorderScores(rawScores)
          };
        });
      }
      formattedMatches.push({
        team1: teams.team,
        team2: teams.opponent,
        displayTime: formatDateTime(matchDate),
        timeUntil: matchStatus.text,
        isLive: matchStatus.isLive,
        isUpcoming: matchStatus.isUpcoming,
        tournament: match.tournament || "Турнир",
        rawDate: matchDate.getTime(),
        bestOf: match.bestof,
        status: match.status,
        finished: match.finished,
        winner: match.winner,
        score: match.score,
        games: gamesInfo,
        currentMap: currentMap,
        seriesScore: seriesScore
      });
    }
  });
  formattedMatches.sort((a, b) => a.rawDate - b.rawDate);
  if (formattedMatches.length === 0) {
    return { hasMatches: false, message: `На данный момент активных матчей ${CONFIG.TEAM_NAME} не найдено`, matches: [] };
  }
  return { hasMatches: true, message: `Найдено матчей: ${formattedMatches.length}`, matches: formattedMatches };
}

function extractMapScores(game, opponents) {
  if (!game || !opponents) return null;
  const scores = [];
  opponents.forEach((opp, index) => {
    if (game.opponents && game.opponents[index]) {
      scores.push({ team: opp.name, score: game.opponents[index].score || 0 });
    }
  });
  return scores;
}

function parseLiquipediaDate(dateString) {
  if (!dateString) return new Date();
  return new Date(dateString.replace(' ', 'T') + 'Z');
}

function extractTeamsFromMatch(match) {
  let team = null, opponent = null;
  const searchName = CONFIG.TEAM_NAME.toLowerCase().trim();
  for (let opp of match.match2opponents) {
    if (!opp || !opp.name) continue;
    const opponentName = opp.name.toLowerCase().trim();
    if (opponentName === searchName) team = opp.name;
    else opponent = opp.name;
  }
  return team && opponent ? { team, opponent } : null;
}

function getMatchStatus(match, hoursDiff) {
  const apiStatus = match.status ? match.status.toLowerCase() : '';
  const isFinished = match.finished === true;
  const hasWinner = match.winner && match.winner !== '';
  const hasLiveGames = match.match2games && Array.isArray(match.match2games) && match.match2games.some(game => !game.finished);
  if (isFinished || hasWinner || apiStatus === 'finished' || apiStatus === 'completed') {
    return { text: "Завершен", isLive: false, isUpcoming: false, isFinished: true };
  }
  if (hasLiveGames || apiStatus === 'live' || apiStatus === 'ongoing') {
    return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
  }
  const minutesDiff = hoursDiff * 60;
  if (minutesDiff >= -180 && minutesDiff <= 5) {
    if (match.score && !isFinished) return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
    if (minutesDiff <= 0 && minutesDiff >= -120) return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
  }
  if (apiStatus === 'upcoming' || apiStatus === 'scheduled') return formatTimeUntil(hoursDiff);
  if (hoursDiff > 0) return formatTimeUntil(hoursDiff);
  return { text: "Завершен (предположительно)", isLive: false, isUpcoming: false, isFinished: true };
}

function formatTimeUntil(hoursDiff) {
  if (hoursDiff > 0) {
    const days = Math.floor(hoursDiff / 24);
    const hours = Math.floor(hoursDiff % 24);
    const minutes = Math.floor((hoursDiff * 60) % 60);
    if (days > 0) return { text: `через ${days}д ${hours}ч`, isLive: false, isUpcoming: true, isFinished: false };
    if (hours > 0) return { text: `через ${hours}ч ${minutes}м`, isLive: false, isUpcoming: true, isFinished: false };
    if (minutes > 5) return { text: `через ${minutes}м`, isLive: false, isUpcoming: true, isFinished: false };
    return { text: "СКОРО", isLive: false, isUpcoming: true, isFinished: false };
  }
  return { text: "Скоро", isLive: false, isUpcoming: true, isFinished: false };
}

function formatDateTime(date) {
  try {
    return new Intl.DateTimeFormat('ru-RU', { timeZone: CONFIG.TIMEZONE, day: '2-digit', month: '2-digit', hour: '2-digit', minute: '2-digit' }).format(date);
  } catch { return date.toISOString(); }
}

function getNextMatch() {
  try {
    const result = getTeamMatches(false);
    if (result.error || !result.hasMatches) return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    const nextMatch = result.matches[0];
    let message = nextMatch.isLive ? `🔴 ONLINE: ${nextMatch.team1} vs ${nextMatch.team2}` : `⏰ Следующий матч: ${nextMatch.team1} vs ${nextMatch.team2} | ${nextMatch.timeUntil}`;
    if (nextMatch.tournament && nextMatch.tournament !== "Турнир") message += ` | ${nextMatch.tournament}`;
    if (nextMatch.bestOf) message += ` | BO${nextMatch.bestOf}`;
    return message;
  } catch (error) {
    console.error('getNextMatch Error:', error);
    return getCachedDataWithFallback();
  }
}

function getAllUpcomingMatches() {
  try {
    const result = getTeamMatches(false);
    if (result.error || !result.hasMatches) return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    const activeMatches = result.matches.filter(m => (m.isLive || m.isUpcoming) && !m.isFinished);
    if (activeMatches.length === 0) return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    let message = `🗓️ Ближайшие матчи ${CONFIG.TEAM_NAME}: `;
    activeMatches.slice(0, 3).forEach((match, index) => {
      const status = match.isLive ? "ONLINE" : match.timeUntil;
      message += `${index + 1}. vs ${match.team2} - ${status}; `;
    });
    if (activeMatches.length > 3) message += `... и еще ${activeMatches.length - 3} матчей`;
    return message;
  } catch (error) {
    console.error('getAllUpcomingMatches Error:', error);
    return getCachedDataWithFallback();
  }
}

function getCachedDataWithFallback() {
  try {
    const cache = CacheService.getScriptCache();
    const rawCached = cache.get("raw_matches_data");
    if (rawCached) {
      const rawCacheData = JSON.parse(rawCached);
      const result = processMatches(rawCacheData.rawMatches);
      if (result.hasMatches && result.matches.length > 0) {
        const nextMatch = result.matches[0];
        return nextMatch.isLive ? `🔴 ONLINE: ${nextMatch.team1} vs ${nextMatch.team2}` : `⏰ Следующий матч: ${nextMatch.team1} vs ${nextMatch.team2} | ${nextMatch.timeUntil}`;
      }
    }
    return "⏳ Данные временно недоступны. Попробуйте позже.";
  } catch { return "❌ Сервис временно недоступен"; }
}

function forceRefresh() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  try {
    console.log('Force refresh requested');
    const allMatches = fetchAllMatchesWithRetry();
    if (allMatches === null) {
      return "⚠️ Не удалось обновить данные. Используются сохраненные данные.";
    }
    const teamMatches = filterTeamMatches(allMatches);
    cache.put("raw_matches_data", JSON.stringify({ rawMatches: teamMatches, timestamp: now, source: 'api_force' }), CONFIG.CACHE_DURATION / 1000);
    cache.put("last_successful_update", now.toString(), 3600);
    return "✅ Данные успешно обновлены";
  } catch (error) {
    console.error('Force refresh failed:', error);
    return "⚠️ Не удалось обновить данные. Используются сохраненные данные.";
  }
}

function fixStaleCache() {
  const cache = CacheService.getScriptCache();
  const currentCache = cache.get("raw_matches_data");
  if (!currentCache) return "❌ Кэш не найден, требуется полное обновление";
  try {
    const cacheData = JSON.parse(currentCache);
    const cacheAgeMinutes = Math.round((new Date().getTime() - cacheData.timestamp) / 60000);
    if (cacheAgeMinutes > 10) {
      console.log('Cache is stale, forcing refresh...');
      return `🔄 Принудительное обновление: ${runBackgroundUpdate()}`;
    } else return `✅ Кэш актуален (${cacheAgeMinutes} минут)`;
  } catch (error) { return `❌ Ошибка: ${error.message}`; }
}

function clearCache() {
  const cache = CacheService.getScriptCache();
  cache.remove("raw_matches_data");
  cache.remove("last_successful_update");
  cache.remove("last_update_attempt");
  cache.remove("last_forced_attempt");
  cache.remove("last_background_attempt");
  cache.remove("last_update_error");
  cache.remove("last_update_error_details");
  cache.remove("last_update_error_time");
  cache.remove("error_count");
  cache.remove("update_in_progress");
  const now = new Date();
  const today = now.toDateString();
  cache.remove(`api_requests_${today}`);
  for (let i = 0; i < 24; i++) cache.remove(`api_requests_${i}`);
  cleanupTriggers();
  return "✅ Кэш очищен";
}

// ================== ВЕБ-ПАНЕЛЬ УПРАВЛЕНИЯ ==================
function getStatusDashboard(shouldRefreshData = false) {
  try {
    if (shouldRefreshData) {
      const cache = CacheService.getScriptCache();
      const now = new Date().getTime();
      const lastUpdate = cache.get("last_successful_update");
      const shouldUpdateForDashboard = !lastUpdate || (now - parseInt(lastUpdate)) > CONFIG.USER_REQUEST_UPDATE_INTERVAL;
      if (shouldUpdateForDashboard) {
        const lastDashboardUpdate = cache.get("last_dashboard_update");
        if (!lastDashboardUpdate || (now - parseInt(lastDashboardUpdate)) > 2 * 60 * 1000) {
          cache.put("last_dashboard_update", now.toString(), 900);
          try { ScriptApp.newTrigger('runBackgroundUpdate').timeBased().after(3000).create(); } catch (triggerError) { }
        }
      }
    }
    const status = getUpdateStatus();
    const cache = CacheService.getScriptCache();
    const currentCache = cache.get("raw_matches_data");
    let matchesInfo = [], matchesError = null;
    if (currentCache) {
      try {
        const cacheData = JSON.parse(currentCache);
        if (cacheData.rawMatches && cacheData.rawMatches.length > 0) {
          const processed = processMatches(cacheData.rawMatches);
          matchesInfo = processed.matches || [];
        }
      } catch (e) { matchesError = 'Ошибка обработки данных матчей: ' + e.toString(); }
    }
    const scriptUrl = ScriptApp.getService().getUrl();
    const html = `<!DOCTYPE html>...`; // полный HTML опущен для краткости, он идентичен предыдущей версии
    return HtmlService.createHtmlOutput(html);
  } catch (error) {
    return HtmlService.createHtmlOutput(`<div style="color:red;">Ошибка: ${error.message}</div>`);
  }
}

// ================== КОМПАКТНЫЙ ОВЕРЛЕЙ ДЛЯ OBS ==================
function getStreamOverlay() {
  try {
    const cache = CacheService.getScriptCache();
    const currentCache = cache.get("raw_matches_data");
    let matchesData = { hasMatches: false, matches: [] };
    if (currentCache) {
      try {
        const cacheData = JSON.parse(currentCache);
        matchesData = processMatches(cacheData.rawMatches);
      } catch (e) {
        console.error('Error parsing cache for overlay:', e);
      }
    }
    const scriptUrl = ScriptApp.getService().getUrl();
    const html = `
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background-color: transparent;
      color: white;
      line-height: 1.3;
      overflow: hidden;
      padding: 0;
      margin: 0;
      text-shadow: 2px 2px 4px rgba(0,0,0,0.8), 0 0 10px rgba(0,0,0,0.5);
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
    }
    .match-card {
      background: transparent;
      padding: 16px 24px;
      display: inline-flex;
      flex-direction: column;
      align-items: center;
      gap: 8px;
    }
    .match-card.live .status-row {
      animation: fire 1.5s infinite alternate;
    }
    .team-row {
      display: grid;
      grid-template-columns: 1fr auto 1fr;
      align-items: center;
      width: 100%;
      font-weight: 700;
      font-size: 1.4em;
      color: white;
    }
    .team-left, .team-right {
      text-align: center;
      white-space: nowrap;
    }
    .vs {
      color: #ff99cc;
      font-size: 1.2em;
      text-shadow: 0 0 10px #ff69b4;
      text-align: center;
      margin: 0 12px;
    }
    .status-row {
      font-size: 1.2em;
      font-weight: 600;
      color: white;
      text-align: center;
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 8px;
    }
    .status-row.live {
      color: #ff99cc;
    }
    .kitty {
      display: inline-block;
      font-size: 1.2em;
      animation: bounce 0.6s infinite alternate ease-in-out;
    }
    .kitty.left {
      animation-delay: 0s;
    }
    .kitty.right {
      animation-delay: 0.3s;
    }
    @keyframes bounce {
      from { transform: translateY(0); }
      to { transform: translateY(-10px); }
    }
    .maps-row {
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
      gap: 12px;
      margin-top: 4px;
      font-size: 1.1em;
    }
    .map-badge {
      background: transparent;
      padding: 4px 12px;
      display: inline-flex;
      align-items: center;
      gap: 6px;
      white-space: nowrap;
    }
    .map-badge.active {
      animation: fire 1.5s infinite alternate;
    }
    .map-number { font-weight: 600; color: #ffb6c1; }
    .map-score { font-weight: 700; color: white; }
    @keyframes fire {
      0% { text-shadow: 0 0 5px #ffb6c1, 0 0 10px #ffb6c1, 0 0 15px #ff69b4; }
      50% { text-shadow: 0 0 10px #ff99cc, 0 0 20px #ff99cc, 0 0 30px #ff1493; }
      100% { text-shadow: 0 0 5px #ffb6c1, 0 0 15px #ff69b4, 0 0 25px #ff1493; }
    }
  </style>
</head>
<body>
  <div id="matchesContainer"></div>
  <script>
    const scriptUrl = "${scriptUrl}";
    const refreshInterval = 15000;

    function renderMatches(matches) {
      if (!matches || matches.length === 0) {
        return '';
      }
      const match = matches[0];
      const isLive = match.isLive;

      const teamHtml = \`
        <div class="team-row">
          <span class="team-left">\${match.team1}</span>
          <span class="vs">vs</span>
          <span class="team-right">\${match.team2}</span>
        </div>
      \`;

      let statusHtml;
      if (isLive) {
        statusHtml = \`
          <div class="status-row live">
            <span class="kitty left">🌸</span>
            <span class="status-text">🔴 LIVE</span>
            <span class="kitty right">🌸</span>
          </div>
        \`;
      } else {
        let timeText = match.timeUntil.replace('через ', '');
        statusHtml = \`
          <div class="status-row">
            <span class="kitty left">🌸</span>
            <span class="status-text">\${timeText}</span>
            <span class="kitty right">🌸</span>
          </div>
        \`;
      }

      let mapsHtml = '';
      if (match.games && match.games.length > 0) {
        mapsHtml = '<div class="maps-row">';
        match.games.forEach((game, idx) => {
          const isActive = match.currentMap && match.currentMap.mapNumber === game.mapNumber;
          const activeClass = isActive ? 'active' : '';
          const scores = game.scores ? game.scores.map(s => s.score).join(':') : '0:0';
          mapsHtml += \`<span class="map-badge \${activeClass}"><span class="map-number">K\${game.mapNumber}</span><span class="map-score">\${scores}</span></span>\`;
        });
        mapsHtml += '</div>';
      } else if (match.currentMap) {
        mapsHtml = '<div class="maps-row">';
        mapsHtml += '<span class="map-badge active"><span class="map-number">K' + match.currentMap.mapNumber + '</span><span class="map-score">' +
          (match.currentMap.scores ? match.currentMap.scores.map(s => s.score).join(':') : '0:0') +
          '</span></span>';
        mapsHtml += '</div>';
      }

      return \`
        <div class="match-card \${isLive ? 'live' : 'upcoming'}" data-raw-date="\${match.rawDate}">
          \${teamHtml}
          \${statusHtml}
          \${mapsHtml}
        </div>
      \`;
    }

    // Функция для обновления данных без пересоздания DOM
    function updateMatchData(newMatch) {
      const card = document.querySelector('.match-card');
      if (!card) return;

      const isLive = newMatch.isLive;

      // Обновляем команды
      const teamLeft = card.querySelector('.team-left');
      const teamRight = card.querySelector('.team-right');
      if (teamLeft) teamLeft.textContent = newMatch.team1;
      if (teamRight) teamRight.textContent = newMatch.team2;

      // Обновляем статусную строку
      const statusRow = card.querySelector('.status-row');
      const statusTextSpan = statusRow?.querySelector('.status-text');
      if (statusRow) {
        if (isLive) {
          statusRow.classList.add('live');
          if (statusTextSpan) {
            statusTextSpan.textContent = '🔴 LIVE';
          } else {
            statusRow.innerHTML = \`
              <span class="kitty left">🌸</span>
              <span class="status-text">🔴 LIVE</span>
              <span class="kitty right">🌸</span>
            \`;
          }
        } else {
          statusRow.classList.remove('live');
          let timeText = newMatch.timeUntil.replace('через ', '');
          if (statusTextSpan) {
            statusTextSpan.textContent = timeText;
          } else {
            statusRow.innerHTML = \`
              <span class="kitty left">🌸</span>
              <span class="status-text">\${timeText}</span>
              <span class="kitty right">🌸</span>
            \`;
          }
        }
      }

      // Обновляем карты
      const mapsRow = card.querySelector('.maps-row');
      if (mapsRow) {
        const mapBadges = mapsRow.querySelectorAll('.map-badge');
        if (newMatch.games && newMatch.games.length > 0) {
          // Если количество карт совпадает, обновляем существующие
          if (newMatch.games.length === mapBadges.length) {
            newMatch.games.forEach((game, idx) => {
              const badge = mapBadges[idx];
              const scoreSpan = badge.querySelector('.map-score');
              if (scoreSpan) {
                const scores = game.scores ? game.scores.map(s => s.score).join(':') : '0:0';
                scoreSpan.textContent = scores;
              }
              const isActive = newMatch.currentMap && newMatch.currentMap.mapNumber === game.mapNumber;
              if (isActive) {
                badge.classList.add('active');
              } else {
                badge.classList.remove('active');
              }
            });
          } else {
            // Если количество изменилось, пересоздаём только секцию карт
            let newMapsHtml = '';
            newMapsHtml = '<div class="maps-row">';
            newMatch.games.forEach((game, idx) => {
              const isActive = newMatch.currentMap && newMatch.currentMap.mapNumber === game.mapNumber;
              const activeClass = isActive ? 'active' : '';
              const scores = game.scores ? game.scores.map(s => s.score).join(':') : '0:0';
              newMapsHtml += \`<span class="map-badge \${activeClass}"><span class="map-number">K\${game.mapNumber}</span><span class="map-score">\${scores}</span></span>\`;
            });
            newMapsHtml += '</div>';
            mapsRow.outerHTML = newMapsHtml;
          }
        } else {
          // Нет карт – удаляем секцию
          mapsRow.remove();
        }
      } else {
        // Если секции карт не было, но они появились – создаём
        if (newMatch.games && newMatch.games.length > 0) {
          let newMapsHtml = '<div class="maps-row">';
          newMatch.games.forEach((game, idx) => {
            const isActive = newMatch.currentMap && newMatch.currentMap.mapNumber === game.mapNumber;
            const activeClass = isActive ? 'active' : '';
            const scores = game.scores ? game.scores.map(s => s.score).join(':') : '0:0';
            newMapsHtml += \`<span class="map-badge \${activeClass}"><span class="map-number">K\${game.mapNumber}</span><span class="map-score">\${scores}</span></span>\`;
          });
          newMapsHtml += '</div>';
          card.appendChild(new DOMParser().parseFromString(newMapsHtml, 'text/html').body.firstChild);
        }
      }

      // Обновляем атрибут data-raw-date для таймера
      card.dataset.rawDate = newMatch.rawDate;
    }

    function updateTimers() {
      const now = new Date().getTime();
      const card = document.querySelector('.match-card');
      if (!card) return;
      const rawDate = card.dataset.rawDate;
      if (!rawDate) return;
      const matchTime = parseInt(rawDate);
      const diff = matchTime - now;
      const statusRow = card.querySelector('.status-row');
      if (!statusRow) return;

      if (diff <= 0) {
        if (!card.classList.contains('live')) {
          card.classList.add('live');
          statusRow.innerHTML = \`
            <span class="kitty left">🌸</span>
            <span class="status-text">🔴 LIVE</span>
            <span class="kitty right">🌸</span>
          \`;
          statusRow.classList.add('live');
        }
      } else {
        const days = Math.floor(diff / (1000 * 60 * 60 * 24));
        const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
        const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
        const seconds = Math.floor((diff % (1000 * 60)) / 1000);
        let displayText = '';
        if (days > 0) displayText = \`\${days}д \${hours}ч\`;
        else if (hours > 0) displayText = \`\${hours}ч \${minutes}м\`;
        else if (minutes > 0) displayText = \`\${minutes}м \${seconds}с\`;
        else displayText = \`\${seconds}с\`;

        const statusTextSpan = statusRow.querySelector('.status-text');
        if (statusTextSpan) {
          statusTextSpan.textContent = displayText;
        } else {
          statusRow.innerHTML = \`
            <span class="kitty left">🌸</span>
            <span class="status-text">\${displayText}</span>
            <span class="kitty right">🌸</span>
          \`;
        }
      }
    }

    async function fetchMatches() {
      try {
        const response = await fetch(scriptUrl + '?function=getMatchesJSON&t=' + Date.now());
        const data = await response.json();
        if (data && data.matches && data.matches.length > 0) {
          const container = document.getElementById('matchesContainer');
          // Если карточка уже существует, обновляем данные без пересоздания
          if (container.querySelector('.match-card')) {
            updateMatchData(data.matches[0]);
          } else {
            // Если карточки нет – создаём впервые
            container.innerHTML = renderMatches(data.matches);
          }
        } else {
          // Нет матчей – очищаем контейнер
          document.getElementById('matchesContainer').innerHTML = '';
        }
      } catch (error) {
        console.error('Error fetching matches:', error);
      }
    }

    setInterval(updateTimers, 1000);
    setInterval(fetchMatches, refreshInterval);

    window.addEventListener('load', () => {
      fetchMatches();
      updateTimers();
    });
  </script>
</body>
</html>
    `;
    return HtmlService.createHtmlOutput(html);
  } catch (error) {
    return HtmlService.createHtmlOutput('');
  }
}

// ================== ОТЛАДКА ==================
function debugMatches() {
  console.log('=== DEBUG START ===');
  const result = getTeamMatches(false);
  console.log('Result:', JSON.stringify(result, null, 2));
  console.log('Next match:', getNextMatch());
  console.log('All matches:', getAllUpcomingMatches());
  const status = getUpdateStatus();
  console.log('Update status:', JSON.stringify(status, null, 2));
  console.log('=== DEBUG END ===');
  return result;
}
