    // Конфигурация
const CONFIG = {
  LIQUIPEDIA_API: "https://api.liquipedia.net/api/v3",
  LIQUIPEDIA_TOKEN: "api-key",
  TEAM_NAME: "PARIVISION",
  CACHE_DURATION: 24 * 60 * 60 * 1000,
  TIMEZONE: "Asia/Krasnoyarsk",
  WIKI: "counterstrike",
  LIMIT: 200,
  
    // интервалы обновления
  BACKGROUND_UPDATE_INTERVAL: 10 * 60 * 1000,
  USER_REQUEST_UPDATE_INTERVAL: 20 * 60 * 1000,
  MIN_UPDATE_INTERVAL: 5 * 60 * 1000,
  
    // Настройки повторных попыток
  MAX_RETRIES: 2,
  RETRY_DELAY: 5000,
  
  ALERT_EMAIL: "",
  SYNC_UPDATE_TIMEOUT: 15000,
  
    // Настройки для AJAX-запросов
  CACHE_AGE_FOR_UPDATE: 10 * 60 * 1000,
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

    // Главные функции для Web API
function doGet(e) {
  const params = e?.parameter || {};
  const functionName = params.function || params.func || 'getNextMatch';
  
  try {
    let result;
    
    // разные типы запросов обрабатываем по-разному
  switch(functionName) {
      case 'getNextMatch':
      case 'getAllUpcomingMatches':
  const updateResult = startGuaranteedUpdate();
        console.log('User request update for', functionName, ':', updateResult);
        break;
        
   case 'getMatchesJSON':
   return ContentService.createTextOutput(JSON.stringify(getTeamMatches(false)))
          .setMimeType(ContentService.MimeType.JSON);
          
   case 'getSystemStatus':
  const cache = CacheService.getScriptCache();
  const currentCache = cache.get("raw_matches_data");
  const now = new Date().getTime();
        
   let shouldForceUpdate = false;
        
  if (currentCache) {
          try {
            const cacheData = JSON.parse(currentCache);
            const cacheAge = now - cacheData.timestamp;
            const cacheAgeMinutes = Math.round(cacheAge / 60000);
            
   console.log('Cache age for AJAX:', cacheAgeMinutes, 'minutes');
            
    // Если кэш очень старый (больше 30 минут) - принудительно обновляем
   if (cacheAgeMinutes > 30) {
  console.log('Cache is very old (' + cacheAgeMinutes + ' min), forcing update...');
  shouldForceUpdate = true;
            }
          } catch (e) {
            console.error('Error parsing cache:', e);
          }
        }
        
  if (shouldForceUpdate) {
          try {
            ScriptApp.newTrigger('runForcedBackgroundUpdate')
              .timeBased()
              .after(1000)
              .create();
            logSystemEvent('Запущено экстренное обновление для очень старого кэша', LOG_CONFIG.LOG_LEVELS.WARNING);
          } catch (triggerError) {
            console.error('Failed to create emergency trigger:', triggerError);
          }
        }
        
  result = JSON.stringify(getUpdateStatus());
  return ContentService.createTextOutput(result).setMimeType(ContentService.MimeType.JSON);
        
  case 'updateStatus':
        result = JSON.stringify(getUpdateStatus());
        return ContentService.createTextOutput(result).setMimeType(ContentService.MimeType.JSON);
        
  default:
        const defaultUpdateResult = startGuaranteedUpdate();
        console.log('Default update for', functionName, ':', defaultUpdateResult);
    }
    
    // Выполняем запрошенную функцию
  switch(functionName) {
      case 'getNextMatch':
        result = getNextMatch();
        break;
      case 'getAllUpcomingMatches':
        result = getAllUpcomingMatches();
        break;
      case 'clearCache':
        result = clearCache();
        break;
      case 'forceRefresh':
        result = forceRefresh();
        break;
      case 'runBackgroundUpdate':
        result = runBackgroundUpdate();
        break;
      case 'fixStaleCache':
        result = fixStaleCache();
        break;
      case 'statusDashboard':
        const shouldRefreshData = params.refresh === 'true';
        return getStatusDashboard(shouldRefreshData);
      case 'clearSystemLogs':
        result = clearSystemLogs();
        break;
      default:
        result = getNextMatch();
    }
    
  return ContentService.createTextOutput(result).setMimeType(ContentService.MimeType.TEXT);
  } catch (error) {
    console.error('doGet Error:', error);
    logSystemEvent('doGet Error: ' + error.message, LOG_CONFIG.LOG_LEVELS.ERROR);
    const fallbackResult = getCachedDataWithFallback();
    return ContentService.createTextOutput(fallbackResult).setMimeType(ContentService.MimeType.TEXT);
  }
}

function doPost(e) {
  return doGet(e);
}

function shouldSkipUpdate(updateType = 'background') {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Проверяем, не выполняется ли уже обновление
  const updateInProgress = cache.get("update_in_progress");
  if (updateInProgress === "true") {
    console.log('Update skipped - already in progress');
    return true;
  }
  
    // 2. Получаем возраст кэша
  const currentCache = cache.get("raw_matches_data");
  let cacheAgeMinutes = 999;
  
  if (currentCache) {
    try {
      const cacheData = JSON.parse(currentCache);
      cacheAgeMinutes = Math.round((now - cacheData.timestamp) / 60000);
    } catch (e) {
      console.error('Error parsing cache for age check:', e);
    }
  }
  
  console.log(`Cache age: ${cacheAgeMinutes} minutes, updateType: ${updateType}`);
  
    // 3. Если кэш очень старый (>30 минут), НЕ пропускаем обновление
  if (cacheAgeMinutes > 30) {
   console.log(`Cache is very old (${cacheAgeMinutes} min), allowing update`);
    return false;
  }
  
    // 4. Если кэш старый (15-30 минут), проверяем последнее успешное обновление
  if (cacheAgeMinutes > 15) {
    const lastUpdate = cache.get("last_successful_update");
    const lastUpdateTime = lastUpdate ? parseInt(lastUpdate) : 0;
    const minutesSinceLastUpdate = Math.round((now - lastUpdateTime) / 60000);
    
  if (minutesSinceLastUpdate > 5) {
      console.log(`Cache is old, last update was ${minutesSinceLastUpdate} min ago, allowing update`);
      return false;
    }
  }
  
    // 5. Для свежего кэша проверяем интервалы между попытками
  if (updateType === 'forced') {
    const lastForcedAttempt = cache.get("last_forced_attempt");
    if (lastForcedAttempt && (now - parseInt(lastForcedAttempt)) < 2 * 60 * 1000) {
      console.log('Forced update skipped - too soon');
      return true;
    }
  } else if (updateType === 'background') {
    const lastBackgroundAttempt = cache.get("last_background_attempt");
    if (lastBackgroundAttempt && (now - parseInt(lastBackgroundAttempt)) < 5 * 60 * 1000) {
      console.log('Background update skipped - too soon');
      return true;
    }
  } else if (updateType === 'user') {
    const lastUserAttempt = cache.get("last_update_attempt");
    if (lastUserAttempt && (now - parseInt(lastUserAttempt)) < 2 * 60 * 1000) {
      console.log('User update skipped - too soon');
      return true;
    }
  }
  
  return false;
}

function runForcedBackgroundUpdate() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Обновляем время попытки для принудительных обновлений
  cache.put("last_forced_attempt", now.toString(), 900);
  
    // 2. Проверяем, нужно ли пропустить
  if (shouldSkipUpdate('forced')) {
    logSystemEvent('Принудительное обновление пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.INFO);
    return "🔄 Принудительное обновление пропущено - слишком частый запрос";
  }
  
  console.log('=== FORCED BACKGROUND UPDATE STARTED ===');
  logSystemEvent('Запущено принудительное фоновое обновление', LOG_CONFIG.LOG_LEVELS.INFO);
  
  try {
  cache.put("update_in_progress", "true", 120);
    
    // Проверяем лимиты API
  const limits = checkAPILimit();
  console.log('API limits:', limits);
    
    // Получаем данные с API
  const allMatches = fetchAllMatchesWithRetry();
  const teamMatches = filterTeamMatches(allMatches);
    
  if (!teamMatches) {
    throw new Error('No team matches received');
    }
    
    // Сохраняем данные
  const rawCacheData = {
      rawMatches: teamMatches,
      timestamp: now,
      source: 'forced_background_update',
      matchesCount: teamMatches.length
    };
    
  cache.put("raw_matches_data", JSON.stringify(rawCacheData), CONFIG.CACHE_DURATION / 1000);
  cache.put("last_successful_update", now.toString(), 3600);
    
    // Сбрасываем счетчик ошибок
  cache.put("error_count", "0", 3600);
    
  console.log('Forced update completed. Matches:', teamMatches.length);
  logSystemEvent(`Принудительное обновление завершено. Матчей: ${teamMatches.length}`, LOG_CONFIG.LOG_LEVELS.SUCCESS);
    
  return `✅ Данные обновлены. Матчей: ${teamMatches.length}`;
    
  } catch (error) {
    console.error('Forced update failed:', error);
    logSystemEvent(`Ошибка принудительного обновления: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return `❌ Ошибка: ${error.message}`;
  } finally {
    cache.remove("update_in_progress");
    
    // Удаляем триггер
  deleteBackgroundUpdateTrigger('runForcedBackgroundUpdate');
  }
}

    // Система логирования
function logSystemEvent(message, level = LOG_CONFIG.LOG_LEVELS.INFO) {
  const cache = CacheService.getScriptCache();
  const timestamp = new Date().getTime();
  
  const logEntry = {
    timestamp: timestamp,
    time: new Date().toLocaleString('ru-RU'),
    level: level,
    message: message,
    source: 'system'
  };
  
    // Получаем текущие логи
  let logs = [];
  const storedLogs = cache.get('system_logs');
  if (storedLogs) {
    try {
      logs = JSON.parse(storedLogs);
    } catch (e) {
      console.error('Error parsing system logs:', e);
    }
  }
  
    // Добавляем новую запись
  logs.unshift(logEntry);
  
    // Ограничиваем количество логов
  if (logs.length > LOG_CONFIG.MAX_LOGS) {
    logs = logs.slice(0, LOG_CONFIG.MAX_LOGS);
  }
  
    // Сохраняем обратно
  cache.put('system_logs', JSON.stringify(logs), 3600);
  
  console.log(`[${level.toUpperCase()}] ${message}`);
}

function getSystemLogs(limit = 20) {
  const cache = CacheService.getScriptCache();
  const storedLogs = cache.get('system_logs');
  
  if (!storedLogs) {
    return [];
  }
  
  try {
    const logs = JSON.parse(storedLogs);
    return logs.slice(0, limit);
  } catch (e) {
    console.error('Error parsing system logs:', e);
    return [];
  }
}

function clearSystemLogs() {
  const cache = CacheService.getScriptCache();
  cache.remove('system_logs');
  logSystemEvent('Системные логи очищены', LOG_CONFIG.LOG_LEVELS.INFO);
  return "✅ Системные логи очищены";
}
function startGuaranteedUpdate() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Обновляем время попытки пользовательского запроса
  cache.put("last_update_attempt", now.toString(), 900);
  
    // 2. Проверяем, нужно ли пропустить обновление для пользователя
  if (shouldSkipUpdate('user')) {
    logSystemEvent('Обновление пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.INFO);
    return "🔄 Обновление пропущено - слишком частый запрос";
  }
  
    // 3. Проверяем, можно ли выполнить фоновое обновление
  if (!shouldSkipUpdate('background')) {
    try {
      const result = runSyncUpdateWithTimeout();
      return result;
    } catch (error) {
      console.log('Sync update failed:', error);
      try {
        ScriptApp.newTrigger('runBackgroundUpdate')
          .timeBased()
          .after(2000)
          .create();
        return "🔄 Запущено асинхронное обновление";
      } catch (triggerError) {
        console.error('Failed to create async trigger:', triggerError);
        return "❌ Ошибка запуска обновления";
      }
    }
  } else {
    logSystemEvent('Фоновое обновление пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.INFO);
    return "🔄 Обновление пропущено - слишком частый запрос";
  }
}

    // Синхронное обновление с таймаутом
function runSyncUpdateWithTimeout() {
  const cache = CacheService.getScriptCache();
  const startTime = new Date().getTime();
  
  try {
  cache.put("update_in_progress", "true", 120);
    
    // Выполняем обновление
  const result = runBackgroundUpdate();
    
    // Убираем флаг
  cache.remove("update_in_progress");
    
  const duration = new Date().getTime() - startTime;
  console.log(`Sync update completed in ${duration}ms: ${result}`);
  logSystemEvent(`Синхронное обновление завершено за ${duration}мс: ${result}`, LOG_CONFIG.LOG_LEVELS.INFO);
  return result;
  } catch (error) {
    cache.remove("update_in_progress");
    throw error;
  }
}

    // Защита от превышения лимитов API
function checkAPILimit() {
  const cache = CacheService.getScriptCache();
  const now = new Date();
  const today = now.toDateString();
  
  const dailyKey = `api_requests_${today}`;
  const hourlyKey = `api_requests_${now.getHours()}`;
  
    // Получаем текущие счетчики
  const dailyCount = parseInt(cache.get(dailyKey) || "0");
  const hourlyCount = parseInt(cache.get(hourlyKey) || "0");
  
    // Увеличиваем счетчики
  cache.put(dailyKey, (dailyCount + 1).toString(), 86400);
  cache.put(hourlyKey, (hourlyCount + 1).toString(), 3600);
  
  console.log(`API requests - Today: ${dailyCount + 1}, This hour: ${hourlyCount + 1}`);
  
    // Разумные лимиты для Liquipedia API
  const DAILY_LIMIT = 800;
  const HOURLY_LIMIT = 100;
  
  if (dailyCount >= DAILY_LIMIT) {
    const error = new Error(`Daily API limit reached: ${dailyCount}/${DAILY_LIMIT}`);
    logSystemEvent(`Достигнут дневной лимит API: ${dailyCount}/${DAILY_LIMIT}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    throw error;
  }
  
  if (hourlyCount >= HOURLY_LIMIT) {
    const error = new Error(`Hourly API limit reached: ${hourlyCount}/${HOURLY_LIMIT}`);
    logSystemEvent(`Достигнут часовой лимит API: ${hourlyCount}/${HOURLY_LIMIT}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    throw error;
  }
  
  return {
    daily: dailyCount + 1,
    hourly: hourlyCount + 1,
    dailyLimit: DAILY_LIMIT,
    hourlyLimit: HOURLY_LIMIT
  };
}

    // Расширенный мониторинг статуса с информацией об API
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
  
    // Мониторинг использования API
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
    } catch (e) {
      cacheInfo = { error: 'Cannot parse cache' };
    }
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
    updateInterval: CONFIG.BACKGROUND_UPDATE_INTERVAL / 60000 + ' minutes',
    backgroundUpdateEnabled: true,
    updateInProgress: updateInProgress === "true",
    cacheInfo: cacheInfo,
    apiUsage: {
      daily: dailyCount,
      hourly: hourlyCount,
      dailyLimit: 800,
      hourlyLimit: 100,
      usagePercent: usagePercent
    },
    updateIntervals: {
      background: CONFIG.BACKGROUND_UPDATE_INTERVAL / 60000 + ' min',
      userRequest: CONFIG.USER_REQUEST_UPDATE_INTERVAL / 60000 + ' min',
      minUpdate: CONFIG.MIN_UPDATE_INTERVAL / 60000 + ' min',
      cacheAgeForUpdate: CONFIG.CACHE_AGE_FOR_UPDATE / 60000 + ' min'
    },
    systemLogs: getSystemLogs(15)
  };
}

    // логирование ошибок
function logError(error, context = 'unknown') {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
  const errorDetails = {
    message: error.message || error.toString(),
    context: context,
    timestamp: now,
    stack: error.stack || 'No stack trace'
  };
  
    // Сохраняем последнюю ошибку
  cache.put("last_update_error", error.message || error.toString(), 3600);
  cache.put("last_update_error_details", JSON.stringify(errorDetails), 3600);
  cache.put("last_update_error_time", now.toString(), 3600);
  
    // Увеличиваем счетчик ошибок
  const currentCount = parseInt(cache.get("error_count") || "0");
  cache.put("error_count", (currentCount + 1).toString(), 3600);
  
    // Логируем в системные логи
  logSystemEvent(`Ошибка в ${context}: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
  
  console.error(`Error [${context}]:`, error);
}

function runBackgroundUpdate() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Обновляем время попытки фонового обновления
  cache.put("last_background_attempt", now.toString(), 900);
  
    // 2. Проверяем, нужно ли пропустить
  if (shouldSkipUpdate('background')) {
    logSystemEvent('Фоновое обновление пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.INFO);
    return "🔄 Обновление пропущено - слишком частый запрос";
  }
  
  const lock = LockService.getScriptCache();
  
  try {
    if (!lock.tryLock(10000)) {
      console.log('Background update skipped - lock not acquired');
      logSystemEvent('Фоновое обновление пропущено - блокировка не получена', LOG_CONFIG.LOG_LEVELS.WARNING);
      return "🔄 Обновление уже выполняется";
    }
    
  console.log('Background update started...');
  logSystemEvent('Запуск фонового обновления данных', LOG_CONFIG.LOG_LEVELS.INFO);
    
  cache.put("update_in_progress", "true", 120);
    
    // Проверка лимитов API
  const limits = checkAPILimit();
  console.log('API limits:', limits);
  logSystemEvent(`Проверка лимитов API пройдена: ${limits.daily}/${limits.dailyLimit} сегодня, ${limits.hourly}/${limits.hourlyLimit} в этом часу`, LOG_CONFIG.LOG_LEVELS.INFO);
    
    // Получаем данные
  const allMatches = fetchAllMatchesWithRetry();
  const teamMatches = filterTeamMatches(allMatches);
    
  if (!teamMatches) {
      throw new Error('No team matches received');
    }
    
    // Сохраняем данные
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
    try {
      lock.releaseLock();
    } catch (e) {
      console.log('Lock release error:', e);
    }
    
  cache.remove("update_in_progress");
  deleteBackgroundUpdateTrigger('runBackgroundUpdate');
  }
}

function fetchAllMatchesWithDebug() {
  const limits = checkAPILimit();
  console.log('API limits check:', limits);
  
  const now = new Date();
  
  const twoHoursAgo = new Date(now.getTime() - 2 * 60 * 60 * 1000);
  const oneMonthFromNow = new Date(now.getTime() + 30 * 24 * 60 * 60 * 1000);
  
  const formatDate = (date) => date.toISOString().split('T')[0];
  const dateFrom = formatDate(twoHoursAgo);
  const dateTo = formatDate(oneMonthFromNow);
  
  const conditions = encodeURIComponent(
    `[[date::>${dateFrom}]] AND [[date::<${dateTo}]]`
  );
  
  const fields = 'date,match2opponents,tournament,status,finished,winner,score,bestof';
  const url = `${CONFIG.LIQUIPEDIA_API}/match?wiki=${CONFIG.WIKI}&limit=${CONFIG.LIMIT}&conditions=${conditions}&fields=${fields}&order=date ASC`;
  
  console.log('Fetching from Liquipedia API:', url);
  logSystemEvent(`Запрос к Liquipedia API: ${url}`, LOG_CONFIG.LOG_LEVELS.DEBUG);
  
  const options = {
    'method': 'GET',
    'headers': {
      'Authorization': `Apikey ${CONFIG.LIQUIPEDIA_TOKEN}`,
      'Accept-Encoding': 'gzip',
      'User-Agent': 'Team Matches Bot/2.0',
      'Accept': 'application/json'
    },
    'muteHttpExceptions': true,
    'validateHttpsCertificates': false
  };
  
  try {
    const response = UrlFetchApp.fetch(url, options);
    const responseCode = response.getResponseCode();
    const contentText = response.getContentText();
    
  if (responseCode !== 200) {
      const error = new Error(`Liquipedia API ошибка: ${responseCode}`);
      logSystemEvent(`Ошибка API Liquipedia: ${responseCode}`, LOG_CONFIG.LOG_LEVELS.ERROR);
      throw error;
    }
    
  const data = JSON.parse(contentText);
    
  if (!data || !data.result) {
      const error = new Error('Некорректный ответ от API');
      logSystemEvent('Некорректный ответ от API Liquipedia', LOG_CONFIG.LOG_LEVELS.ERROR);
      throw error;
    }
    
  console.log('API returned', data.result.length, 'matches');
  logSystemEvent(`API вернул ${data.result.length} матчей`, LOG_CONFIG.LOG_LEVELS.INFO);
  return data.result;
  } catch (error) {
    console.error('API fetch error:', error);
    logSystemEvent(`Ошибка запроса к API: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    throw error;
  }
}

    // Улучшенная функция повторных попыток
function fetchAllMatchesWithRetry() {
  let lastError;
  
  for (let attempt = 1; attempt <= CONFIG.MAX_RETRIES; attempt++) {
    try {
      console.log(`API attempt ${attempt} of ${CONFIG.MAX_RETRIES}`);
      logSystemEvent(`Попытка API ${attempt} из ${CONFIG.MAX_RETRIES}`, LOG_CONFIG.LOG_LEVELS.INFO);
      
  const result = fetchAllMatchesWithDebug();
      
  if (!result || !Array.isArray(result)) {
        throw new Error('Invalid API response format');
      }
      
  console.log(`API attempt ${attempt} successful, matches: ${result.length}`);
      logSystemEvent(`Попытка API ${attempt} успешна, матчей: ${result.length}`, LOG_CONFIG.LOG_LEVELS.SUCCESS);
      return result;
      
  } catch (error) {
      lastError = error;
      console.error(`API attempt ${attempt} failed:`, error);
      logSystemEvent(`Попытка API ${attempt} не удалась: ${error.message}`, LOG_CONFIG.LOG_LEVELS.WARNING);
      
  if (attempt < CONFIG.MAX_RETRIES) {
        const delay = CONFIG.RETRY_DELAY * attempt;
        console.log(`Waiting ${delay}ms before retry...`);
        logSystemEvent(`Ожидание ${delay}мс перед повторной попыткой`, LOG_CONFIG.LOG_LEVELS.INFO);
        Utilities.sleep(delay);
      }
    }
  }
  
    // Если все попытки провалились - возвращаем пустой массив вместо ошибки
  console.error(`All ${CONFIG.MAX_RETRIES} API attempts failed, returning empty array`);
  logSystemEvent(`Все ${CONFIG.MAX_RETRIES} попытки API не удались, возвращаем пустой массив`, LOG_CONFIG.LOG_LEVELS.ERROR);
  return [];
}

    // Функция для удаления триггеров
function deleteBackgroundUpdateTrigger(functionName = 'runBackgroundUpdate') {
  try {
    const triggers = ScriptApp.getProjectTriggers();
    for (const trigger of triggers) {
      if (trigger.getHandlerFunction() === functionName) {
        ScriptApp.deleteTrigger(trigger);
        console.log(`${functionName} trigger deleted`);
        logSystemEvent(`Триггер ${functionName} удален`, LOG_CONFIG.LOG_LEVELS.INFO);
      }
    }
  } catch (error) {
    console.error('Error deleting trigger:', error);
    logSystemEvent(`Ошибка удаления триггера: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
  }
}

    // Функция для получения данных из кэша
function getTeamMatches(shouldUpdate = false) {
  const cache = CacheService.getScriptCache();
  const rawCached = cache.get("raw_matches_data");
  
    // Обновляем данные только если явно запрошено
  if (shouldUpdate) {
    console.log('Forced update requested, fetching fresh data...');
    logSystemEvent('Запрошено принудительное обновление данных', LOG_CONFIG.LOG_LEVELS.INFO);
    return fetchDataSynchronously();
  }
  
  if (rawCached) {
    try {
      const rawCacheData = JSON.parse(rawCached);
      const ageMinutes = Math.round((new Date().getTime() - rawCacheData.timestamp) / 60000);
      console.log('Using cached data, age:', ageMinutes + 'min');
      logSystemEvent(`Используются кэшированные данные, возраст: ${ageMinutes} мин`, LOG_CONFIG.LOG_LEVELS.INFO);
      
      // Обрабатываем сырые данные с ТЕКУЩИМ временем
  const result = processMatches(rawCacheData.rawMatches);
      return result;
    } catch (e) {
      console.error('Error parsing raw cache, will try to refresh', e);
      logSystemEvent(`Ошибка парсинга кэша, пробуем обновить: ${e.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    }
  }
  
  console.log('No raw cache available, fetching data synchronously');
  logSystemEvent('Кэш не найден, синхронное получение данных', LOG_CONFIG.LOG_LEVELS.WARNING);
  return fetchDataSynchronously();
}

    // Синхронное получение данных (только когда кэша нет)
function fetchDataSynchronously() {
  try {
    console.log('Synchronous API fetch started');
    logSystemEvent('Запуск синхронного получения данных', LOG_CONFIG.LOG_LEVELS.INFO);
    
  const allMatches = fetchAllMatchesWithRetry();
  const teamMatches = filterTeamMatches(allMatches);
  const result = processMatches(teamMatches);
    
    // Сохраняем СЫРЫЕ данные в кэш
  const rawCacheData = {
      rawMatches: teamMatches,
      timestamp: new Date().getTime(),
      source: 'api_sync'
    };
    const cache = CacheService.getScriptCache();
    cache.put("raw_matches_data", JSON.stringify(rawCacheData), CONFIG.CACHE_DURATION / 1000);
    cache.put("last_successful_update", new Date().getTime().toString(), 3600);
    
  console.log('Synchronous fetch completed successfully');
  logSystemEvent('Синхронное получение данных завершено успешно', LOG_CONFIG.LOG_LEVELS.SUCCESS);
    
  return result;
  } catch (error) {
    console.error('Synchronous fetch failed:', error);
    logSystemEvent(`Синхронное получение данных не удалось: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return {
      hasMatches: false,
      message: `Данные временно недоступны. Попробуйте обновить через несколько минут.`,
      matches: [],
      cacheUsed: false,
      apiError: true
    };
  }
}

    // Фильтрация матчей по команде
function filterTeamMatches(matches) {
  if (!matches || !Array.isArray(matches)) {
    return [];
  }
  
  console.log('Filtering', matches.length, 'matches for team:', CONFIG.TEAM_NAME);
  logSystemEvent(`Фильтрация ${matches.length} матчей для команды: ${CONFIG.TEAM_NAME}`, LOG_CONFIG.LOG_LEVELS.INFO);
  
  const searchName = CONFIG.TEAM_NAME.toLowerCase().trim();
  
  const filtered = matches.filter(match => {
    if (!match || !match.match2opponents) return false;
    
  for (let opponent of match.match2opponents) {
      if (opponent && opponent.name) {
        const opponentName = opponent.name.toLowerCase().trim();
        if (opponentName === searchName) {
          return true;
        }
      }
    }
    return false;
  });
  
  console.log('Found', filtered.length, 'matches for', CONFIG.TEAM_NAME);
  logSystemEvent(`Найдено ${filtered.length} матчей для ${CONFIG.TEAM_NAME}`, LOG_CONFIG.LOG_LEVELS.INFO);
  return filtered;
}

    // Обработка матчей - ВСЕГДА с текущим временем
function processMatches(matches) {
  if (!matches || matches.length === 0) {
    logSystemEvent(`Матчи для ${CONFIG.TEAM_NAME} не найдены`, LOG_CONFIG.LOG_LEVELS.INFO);
    return {
      hasMatches: false,
      message: `На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`,
      matches: []
    };
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
    
    // Показываем матчи, которые еще не начались или начались не более 4 часов назад
  if (hoursDiff < -4) return;
    
  const matchStatus = getMatchStatus(match, hoursDiff);
    
  if (!matchStatus.isFinished) {
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
        score: match.score
      });
    }
  });
  
  formattedMatches.sort((a, b) => a.rawDate - b.rawDate);
  
  if (formattedMatches.length === 0) {
    logSystemEvent(`Активных матчей для ${CONFIG.TEAM_NAME} не найдено`, LOG_CONFIG.LOG_LEVELS.INFO);
    return {
      hasMatches: false,
      message: `На данный момент активных матчей ${CONFIG.TEAM_NAME} не найдено`,
      matches: []
    };
  }
  
  logSystemEvent(`Обработано ${formattedMatches.length} матчей для ${CONFIG.TEAM_NAME}`, LOG_CONFIG.LOG_LEVELS.INFO);
  return {
    hasMatches: true,
    message: `Найдено матчей: ${formattedMatches.length}`,
    matches: formattedMatches
  };
}

    // Парсинг даты из Liquipedia
function parseLiquipediaDate(dateString) {
  if (!dateString) return new Date();
  const utcDateString = dateString.replace(' ', 'T') + 'Z';
  return new Date(utcDateString);
}

    // Извлечение команд из матча
function extractTeamsFromMatch(match) {
  let team = null;
  let opponent = null;
  
  const searchName = CONFIG.TEAM_NAME.toLowerCase().trim();
  
  for (let opp of match.match2opponents) {
    if (!opp || !opp.name) continue;
    const opponentName = opp.name.toLowerCase().trim();
    
   if (opponentName === searchName) {
      team = opp.name;
    } else {
      opponent = opp.name;
    }
  }
  
  return team && opponent ? { team, opponent } : null;
}

    // Функция определения статуса матча
function getMatchStatus(match, hoursDiff) {
  const apiStatus = match.status ? match.status.toLowerCase() : '';
  const isFinished = match.finished === true;
  const hasWinner = match.winner && match.winner !== '';
  const hasScore = match.score && match.score !== '';
  
  if (isFinished || hasWinner || apiStatus === 'finished' || apiStatus === 'completed') {
    return { text: "Завершен", isLive: false, isUpcoming: false, isFinished: true };
  }
  
    // ЛОГИКА ДЛЯ LIVE МАТЧЕЙ
  if (apiStatus === 'live' || apiStatus === 'ongoing') {
    return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
  }
  
    // МАТЧИ, КОТОРЫЕ ДОЛЖНЫ БЫТЬ LIVE, НО API ЕЩЁ НЕ ОБНОВИЛОСЬ
  const minutesDiff = hoursDiff * 60;
  if (minutesDiff >= -180 && minutesDiff <= 5) {
    if (hasScore && !isFinished) {
      return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
    }
    
    // Если матч должен идти по расписанию, но статус не обновлен
  if (minutesDiff <= 0 && minutesDiff >= -120) {
      return { text: "🔴 ONLINE СЕЙЧАС", isLive: true, isUpcoming: false, isFinished: false };
    }
  }
  
  if (apiStatus === 'upcoming' || apiStatus === 'scheduled') {
    return formatTimeUntil(hoursDiff);
  }
  
  if (hoursDiff > 0) {
    return formatTimeUntil(hoursDiff);
  } else {
    return { text: "Завершен (предположительно)", isLive: false, isUpcoming: false, isFinished: true };
  }
}

    // Форматирование времени до матча - ВСЕГДА актуальное
function formatTimeUntil(hoursDiff) {
  if (hoursDiff > 0) {
    const days = Math.floor(hoursDiff / 24);
    const hours = Math.floor(hoursDiff % 24);
    const minutes = Math.floor((hoursDiff * 60) % 60);
    
   if (days > 0) {
      return { text: `через ${days}д ${hours}ч`, isLive: false, isUpcoming: true, isFinished: false };
    } else if (hours > 0) {
      return { text: `через ${hours}ч ${minutes}м`, isLive: false, isUpcoming: true, isFinished: false };
    } else if (minutes > 5) {
      return { text: `через ${minutes}м`, isLive: false, isUpcoming: true, isFinished: false };
    } else {
      return { text: "СКОРО", isLive: false, isUpcoming: true, isFinished: false };
    }
  }
  
  return { text: "Скоро", isLive: false, isUpcoming: true, isFinished: false };
}

    // Форматирование даты и времени
function formatDateTime(date) {
  try {
    const formatter = new Intl.DateTimeFormat('ru-RU', {
      timeZone: CONFIG.TIMEZONE,
      day: '2-digit', month: '2-digit', hour: '2-digit', minute: '2-digit'
    });
    return formatter.format(date);
  } catch (error) {
    return date.toISOString();
  }
}

    // Функция для StreamElements - следующий матч
function getNextMatch() {
  try {
    const result = getTeamMatches(false);
    
  if (result.error) {
      logSystemEvent('Временные проблемы с получением данных для getNextMatch', LOG_CONFIG.LOG_LEVELS.WARNING);
      return "⚠️ Временные проблемы с получением данных";
    }
    
  if (!result.hasMatches) {
      logSystemEvent(`Матчей ${CONFIG.TEAM_NAME} не запланировано (getNextMatch)`, LOG_CONFIG.LOG_LEVELS.INFO);
      return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    }
    
   const nextMatch = result.matches[0];
    let message = nextMatch.isLive ? 
      `🔴 ONLINE: ${nextMatch.team1} vs ${nextMatch.team2}` :
      `⏰ Следующий матч: ${nextMatch.team1} vs ${nextMatch.team2} | ${nextMatch.timeUntil}`;
    
  if (nextMatch.tournament && nextMatch.tournament !== "Турнир") {
      message += ` | ${nextMatch.tournament}`;
    }
    
  if (nextMatch.bestOf) {
      message += ` | BO${nextMatch.bestOf}`;
    }
    
   logSystemEvent(`getNextMatch: ${message}`, LOG_CONFIG.LOG_LEVELS.INFO);
    return message;
  } catch (error) {
    console.error('getNextMatch Error:', error);
    logSystemEvent(`Ошибка getNextMatch: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return getCachedDataWithFallback();
  }
}

    // Функция для StreamElements - все предстоящие матчи
function getAllUpcomingMatches() {
  try {
    const result = getTeamMatches(false);
    
  if (result.error || !result.hasMatches) {
      logSystemEvent(`Матчей ${CONFIG.TEAM_NAME} не запланировано (getAllUpcomingMatches)`, LOG_CONFIG.LOG_LEVELS.INFO);
      return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    }
    
  const activeMatches = result.matches.filter(match => 
      (match.isLive || match.isUpcoming) && !match.isFinished
    );
    
  if (activeMatches.length === 0) {
      logSystemEvent(`Активных матчей ${CONFIG.TEAM_NAME} не найдено (getAllUpcomingMatches)`, LOG_CONFIG.LOG_LEVELS.INFO);
      return `⏳ На данный момент матчей ${CONFIG.TEAM_NAME} не запланировано`;
    }
    
  let message = `🗓️ Ближайшие матчи ${CONFIG.TEAM_NAME}: `;
    activeMatches.slice(0, 3).forEach((match, index) => {
      const status = match.isLive ? "ONLINE" : match.timeUntil;
      message += `${index + 1}. vs ${match.team2} - ${status}; `;
    });
    
  if (activeMatches.length > 3) {
      message += `... и еще ${activeMatches.length - 3} матчей`;
    }
    
  logSystemEvent(`getAllUpcomingMatches: ${message}`, LOG_CONFIG.LOG_LEVELS.INFO);
    return message;
  } catch (error) {
    console.error('getAllUpcomingMatches Error:', error);
    logSystemEvent(`Ошибка getAllUpcomingMatches: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return getCachedDataWithFallback();
  }
}

    // Резервное получение данных при ошибках
function getCachedDataWithFallback() {
  try {
    const cache = CacheService.getScriptCache();
    const rawCached = cache.get("raw_matches_data");
    
  if (rawCached) {
      const rawCacheData = JSON.parse(rawCached);
      const cacheAge = Math.round((new Date().getTime() - rawCacheData.timestamp) / 60000);
      
    // Обрабатываем сырые данные с ТЕКУЩИМ временем
  const result = processMatches(rawCacheData.rawMatches);
      
  if (result.hasMatches && result.matches.length > 0) {
        const nextMatch = result.matches[0];
        logSystemEvent('Использованы кэшированные данные (fallback)', LOG_CONFIG.LOG_LEVELS.WARNING);
        return nextMatch.isLive ? 
          `🔴 ONLINE: ${nextMatch.team1} vs ${nextMatch.team2}` :
          `⏰ Следующий матч: ${nextMatch.team1} vs ${nextMatch.team2} | ${nextMatch.timeUntil}`;
      }
    }
    
   logSystemEvent('Данные временно недоступны (fallback)', LOG_CONFIG.LOG_LEVELS.WARNING);
    return "⏳ Данные временно недоступны. Попробуйте позже.";
  } catch (error) {
    logSystemEvent('Сервис временно недоступен (fallback)', LOG_CONFIG.LOG_LEVELS.ERROR);
    return "❌ Сервис временно недоступен";
  }
}

    // ФУНКЦИЯ ПРИНУДИТЕЛЬНОГО ОБНОВЛЕНИЯ
function forceRefresh() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Обновляем время попытки для принудительных обновлений
  cache.put("last_forced_attempt", now.toString(), 900);
  
    // 2. Проверяем, нужно ли пропустить
  if (shouldSkipUpdate('forced')) {
    logSystemEvent('Принудительное обновление пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.WARNING);
    return "🔄 Принудительное обновление пропущено - слишком частый запрос";
  }
  
  try {
    console.log('Force refresh requested');
    logSystemEvent('Запрос принудительного обновления', LOG_CONFIG.LOG_LEVELS.INFO);
    
  const allMatches = fetchAllMatchesWithRetry();
  const teamMatches = filterTeamMatches(allMatches);
    
  const rawCacheData = {
      rawMatches: teamMatches,
      timestamp: now,
      source: 'api_force'
    };
    
  cache.put("raw_matches_data", JSON.stringify(rawCacheData), CONFIG.CACHE_DURATION / 1000);
  cache.put("last_successful_update", now.toString(), 3600);
    
  console.log('Force refresh completed - raw data updated');
  logSystemEvent('Принудительное обновление завершено - данные обновлены', LOG_CONFIG.LOG_LEVELS.SUCCESS);
    
  return "✅ Данные успешно обновлены";
  } catch (error) {
    console.error('Force refresh failed:', error);
    logSystemEvent(`Принудительное обновление не удалось: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return "⚠️ Не удалось обновить данные. Используются сохраненные данные.";
  }
}

    // ФУНКЦИЯ ИСПРАВЛЕНИЯ КЭША
function fixStaleCache() {
  const cache = CacheService.getScriptCache();
  const now = new Date().getTime();
  
    // 1. Обновляем время попытки фонового обновления
  cache.put("last_background_attempt", now.toString(), 900);
  
    // 2. Проверяем, нужно ли пропустить
  if (shouldSkipUpdate('background')) {
    logSystemEvent('Исправление кэша пропущено - слишком частый запрос', LOG_CONFIG.LOG_LEVELS.WARNING);
    return "🔄 Исправление кэша пропущено - слишком частый запрос";
  }
  
  const currentCache = cache.get("raw_matches_data");
  
  if (!currentCache) {
    logSystemEvent('Кэш не найден, требуется полное обновление', LOG_CONFIG.LOG_LEVELS.WARNING);
    return "❌ Кэш не найден, требуется полное обновление";
  }
  
  try {
    const cacheData = JSON.parse(currentCache);
    const cacheAge = now - cacheData.timestamp;
    const cacheAgeMinutes = Math.round(cacheAge / 60000);
    
  console.log(`Cache age: ${cacheAgeMinutes} minutes, forcing refresh`);
  logSystemEvent(`Возраст кэша: ${cacheAgeMinutes} минут, принудительное обновление`, LOG_CONFIG.LOG_LEVELS.INFO);
    
  if (cacheAgeMinutes > 10) {
      console.log('Cache is stale, forcing refresh...');
      logSystemEvent('Кэш устарел, запуск принудительного обновления', LOG_CONFIG.LOG_LEVELS.WARNING);
      const result = runBackgroundUpdate();
      return `🔄 Принудительное обновление: ${result}`;
    } else {
      logSystemEvent(`Кэш актуален (${cacheAgeMinutes} минут)`, LOG_CONFIG.LOG_LEVELS.INFO);
      return `✅ Кэш актуален (${cacheAgeMinutes} минут)`;
    }
  } catch (error) {
    logSystemEvent(`Ошибка проверки кэша: ${error.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
    return `❌ Ошибка: ${error.message}`;
  }
}

    // Очистка кэша
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
  
    // Очищаем счетчики API
  const now = new Date();
  const today = now.toDateString();
  cache.remove(`api_requests_${today}`);
  for (let i = 0; i < 24; i++) {
    cache.remove(`api_requests_${i}`);
  }
  
    // Удаляем старые триггеры если есть
  const triggers = ScriptApp.getProjectTriggers();
  for (const trigger of triggers) {
    if (trigger.getHandlerFunction() === 'runBackgroundUpdate' || trigger.getHandlerFunction() === 'runForcedBackgroundUpdate') {
      ScriptApp.deleteTrigger(trigger);
    }
  }
  
  console.log("✅ Кэш очищен");
  logSystemEvent('Кэш очищен', LOG_CONFIG.LOG_LEVELS.INFO);
  return "✅ Кэш очищен";
}

    // ВЕБ-ПАНЕЛЬ
function getStatusDashboard(shouldRefreshData = false) {
  try {
    if (shouldRefreshData) {
      const cache = CacheService.getScriptCache();
      const now = new Date().getTime();
      const lastUpdate = cache.get("last_successful_update");
      
      // Проверяем, не было ли успешного обновления в последние 2 минуты
  const shouldUpdateForDashboard = !lastUpdate || 
        (now - parseInt(lastUpdate)) > CONFIG.USER_REQUEST_UPDATE_INTERVAL;
      
  if (shouldUpdateForDashboard) {
        logSystemEvent('Панель управления запросила обновление данных', LOG_CONFIG.LOG_LEVELS.INFO);
        
      // Используем отдельный флаг для обновлений с панели
  const lastDashboardUpdate = cache.get("last_dashboard_update");
        if (!lastDashboardUpdate || (now - parseInt(lastDashboardUpdate)) > 2 * 60 * 1000) {
          cache.put("last_dashboard_update", now.toString(), 900);
          
      // Запускаем в фоне, без ожидания
  try {
            ScriptApp.newTrigger('runBackgroundUpdate')
              .timeBased()
              .after(3000)
              .create();
          } catch (triggerError) {
            console.error('Failed to create dashboard trigger:', triggerError);
          }
        }
      }
    }
    
  const status = getUpdateStatus();
  const cache = CacheService.getScriptCache();
  const currentCache = cache.get("raw_matches_data");
    
  let matchesInfo = [];
  let matchesError = null;
    
  if (currentCache) {
      try {
        const cacheData = JSON.parse(currentCache);
        if (cacheData.rawMatches && cacheData.rawMatches.length > 0) {
          const processed = processMatches(cacheData.rawMatches);
          matchesInfo = processed.matches || [];
        }
      } catch (e) {
        console.error('Error parsing cache for dashboard:', e);
        matchesError = 'Ошибка обработки данных матчей: ' + e.toString();
        logSystemEvent(`Ошибка парсинга кэша для дашборда: ${e.message}`, LOG_CONFIG.LOG_LEVELS.ERROR);
      }
    }
    
  const scriptUrl = ScriptApp.getService().getUrl();
    
  const html = `
<!DOCTYPE html>
<html>
<head>
  <title>Мониторинг матчей ${CONFIG.TEAM_NAME}</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    :root {
      --primary-color: #3498db;
      --success-color: #2ecc71;
      --warning-color: #f39c12;
      --danger-color: #e74c3c;
      --dark-color: #2c3e50;
      --light-color: #ecf0f1;
      --text-color: #333;
      --border-radius: 12px;
      --box-shadow: 0 8px 25px rgba(0,0,0,0.1);
      --transition: all 0.3s ease;
    }
    
  * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    
    body { 
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
      margin: 0; 
      padding: 0;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      color: var(--text-color);
      line-height: 1.6;
    }
    
    .container {
      max-width: 1400px;
      margin: 0 auto;
      padding: 20px;
    }
    
    .header {
      background: linear-gradient(135deg, var(--dark-color), #34495e);
      color: white;
      padding: 30px;
      border-radius: var(--border-radius);
      text-align: center;
      margin-bottom: 25px;
      box-shadow: var(--box-shadow);
      position: relative;
      overflow: hidden;
    }
    
    .header::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 4px;
      background: linear-gradient(90deg, var(--primary-color), var(--success-color), var(--warning-color));
    }
    
    .header h1 {
      margin: 0;
      font-size: 2.8em;
      font-weight: 700;
      letter-spacing: -0.5px;
    }
    
    .header .subtitle {
      opacity: 0.9;
      margin-top: 8px;
      font-size: 1.1em;
      font-weight: 300;
    }
    
    .team-badge {
      display: inline-block;
      background: rgba(255,255,255,0.2);
      padding: 8px 20px;
      border-radius: 50px;
      margin-top: 15px;
      font-weight: 600;
      backdrop-filter: blur(10px);
    }
    
        /* СЕКЦИЯ МАТЧЕЙ - ПЕРЕНЕСЕНА ВВЕРХ */
    .matches-section {
      margin-bottom: 30px;
    }
    
    .section-title {
      font-size: 1.5em;
      font-weight: 600;
      margin-bottom: 20px;
      color: white;
      display: flex;
      align-items: center;
      gap: 10px;
    }
    
    .section-title::before {
      content: '';
      width: 4px;
      height: 24px;
      background: var(--primary-color);
      border-radius: 2px;
    }
    
    .matches-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
      gap: 20px;
      margin-bottom: 10px;
    }
    
    .match-card {
      background: white;
      border-radius: var(--border-radius);
      padding: 20px;
      box-shadow: var(--box-shadow);
      transition: var(--transition);
      border-left: 5px solid var(--primary-color);
      position: relative;
      overflow: hidden;
    }
    
    .match-card:hover {
      transform: translateY(-5px);
      box-shadow: 0 12px 30px rgba(0,0,0,0.15);
    }
    
    .match-card.live {
      border-left-color: var(--danger-color);
      background: linear-gradient(135deg, #fff, #fff5f5);
    }
    
    .match-card.upcoming {
      border-left-color: var(--warning-color);
      background: linear-gradient(135deg, #fff, #fffaf0);
    }
    
    .match-header {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      margin-bottom: 15px;
    }
    
    .match-teams {
      flex: 1;
    }
    
    .team-vs {
      font-size: 1.2em;
      font-weight: 600;
      margin-bottom: 5px;
      color: var(--dark-color);
    }
    
    .match-status {
      padding: 6px 16px;
      border-radius: 20px;
      font-size: 0.85em;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 0.5px;
    }
    
    .status-live {
      background: var(--danger-color);
      color: white;
      animation: pulse 2s infinite;
    }
    
    .status-upcoming {
      background: var(--warning-color);
      color: white;
    }
    
    @keyframes pulse {
      0% { opacity: 1; }
      50% { opacity: 0.7; }
      100% { opacity: 1; }
    }
    
    .match-details {
      display: grid;
      grid-template-columns: 1fr auto;
      gap: 15px;
      align-items: center;
    }
    
    .match-tournament {
      display: flex;
      align-items: center;
      gap: 8px;
      color: #666;
      font-size: 0.95em;
    }
    
    .match-time {
      text-align: right;
    }
    
    .time-main {
      font-size: 1.1em;
      font-weight: 600;
      color: var(--dark-color);
      transition: all 0.3s ease;
    }
    
    .time-main.live-now {
      color: #e74c3c;
      font-weight: bold;
      animation: pulse 2s infinite;
    }
    
    .time-main.upcoming-time {
      color: #f39c12;
      font-weight: 600;
    }
    
    .time-secondary {
      font-size: 0.9em;
      color: #777;
      margin-top: 4px;
    }
    
    .match-meta {
      display: flex;
      gap: 15px;
      margin-top: 12px;
      padding-top: 12px;
      border-top: 1px solid #eee;
    }
    
    .meta-item {
      display: flex;
      align-items: center;
      gap: 6px;
      font-size: 0.9em;
      color: #666;
    }
    
        /* СЕТКА СТАТИСТИКИ */
    .stats-section {
      margin-bottom: 30px;
    }
    
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 20px;
      margin-bottom: 20px;
    }
    
    .stat-card {
      background: white;
      border-radius: var(--border-radius);
      padding: 25px;
      box-shadow: var(--box-shadow);
      transition: var(--transition);
      position: relative;
      overflow: hidden;
    }
    
    .stat-card::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 4px;
      background: var(--primary-color);
    }
    
    .stat-card:hover {
      transform: translateY(-3px);
      box-shadow: 0 10px 25px rgba(0,0,0,0.12);
    }
    
    .stat-card.warning::before {
      background: var(--warning-color);
    }
    
    .stat-card.success::before {
      background: var(--success-color);
    }
    
    .stat-card.error::before {
      background: var(--danger-color);
    }
    
    .stat-card h3 {
      margin: 0 0 15px 0;
      color: var(--dark-color);
      font-size: 1.1em;
      font-weight: 600;
      display: flex;
      align-items: center;
      gap: 10px;
    }
    
    .stat-value {
      font-size: 2em;
      font-weight: 700;
      margin: 10px 0;
      color: var(--dark-color);
    }
    
    .stat-details {
      font-size: 0.9em;
      color: #666;
      line-height: 1.5;
    }
    
    .stat-details p {
      margin: 5px 0;
    }
    
    .api-usage-bar {
      background: var(--light-color);
      border-radius: 10px;
      height: 12px;
      margin: 15px 0;
      overflow: hidden;
    }
    
    .api-usage-fill {
      height: 100%;
      background: linear-gradient(90deg, var(--success-color), var(--warning-color), var(--danger-color));
      transition: width 0.5s ease;
      border-radius: 10px;
    }
    
    .error-details {
      background: #ffeaa7;
      padding: 15px;
      border-radius: 8px;
      margin: 15px 0;
      font-family: 'Courier New', monospace;
      font-size: 0.85em;
      border-left: 4px solid var(--warning-color);
    }
    
        /* НОВАЯ СЕКЦИЯ: КОНСОЛЬ ЛОГОВ */
    .console-section {
      margin-bottom: 30px;
    }
    
    .console-container {
      background: #1e272e;
      border-radius: var(--border-radius);
      padding: 20px;
      box-shadow: var(--box-shadow);
      color: #ecf0f1;
      font-family: 'Courier New', monospace;
      font-size: 0.9em;
      max-height: 300px;
      overflow-y: auto;
    }
    
    .console-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 15px;
      padding-bottom: 10px;
      border-bottom: 1px solid #34495e;
    }
    
    .console-title {
      font-size: 1.2em;
      font-weight: 600;
      color: #ecf0f1;
    }
    
    .console-controls {
      display: flex;
      gap: 10px;
    }
    
    .console-btn {
      background: #34495e;
      color: #ecf0f1;
      border: none;
      padding: 6px 12px;
      border-radius: 4px;
      cursor: pointer;
      font-size: 0.8em;
      transition: var(--transition);
    }
    
    .console-btn:hover {
      background: #4a6572;
    }
    
    .log-entry {
      margin-bottom: 8px;
      padding: 5px 0;
      border-bottom: 1px solid #2c3e50;
    }
    
    .log-time {
      color: #7f8c8d;
      font-size: 0.8em;
      margin-right: 10px;
    }
    
    .log-level {
      padding: 2px 6px;
      border-radius: 3px;
      font-size: 0.75em;
      font-weight: bold;
      margin-right: 8px;
    }
    
    .log-level-info {
      background: #3498db;
      color: white;
    }
    
    .log-level-success {
      background: #2ecc71;
      color: white;
    }
    
    .log-level-warning {
      background: #f39c12;
      color: white;
    }
    
    .log-level-error {
      background: #e74c3c;
      color: white;
    }
    
    .log-level-debug {
      background: #9b59b6;
      color: white;
    }
    
    .log-message {
      color: #ecf0f1;
      word-break: break-word;
    }
    
    .no-logs {
      text-align: center;
      color: #7f8c8d;
      font-style: italic;
      padding: 20px;
    }
    
        /* ПАНЕЛЬ УПРАВЛЕНИЯ */
    .controls-section {
      margin-bottom: 20px;
    }
    
    .controls-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 15px;
    }
    
    .control-btn {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
      padding: 15px 20px;
      background: white;
      color: var(--dark-color);
      text-decoration: none;
      border-radius: var(--border-radius);
      font-weight: 600;
      transition: var(--transition);
      box-shadow: var(--box-shadow);
      border: none;
      cursor: pointer;
      font-size: 1em;
    }
    
    .control-btn:hover {
      transform: translateY(-3px);
      box-shadow: 0 10px 25px rgba(0,0,0,0.15);
      background: var(--primary-color);
      color: white;
    }
    
    .btn-success {
      background: var(--success-color);
      color: white;
    }
    
    .btn-warning {
      background: var(--warning-color);
      color: white;
    }
    
    .btn-danger {
      background: var(--danger-color);
      color: white;
    }
    
        /* ФУТЕР */
    .footer {
      text-align: center;
      padding: 20px;
      background: rgba(255,255,255,0.1);
      border-radius: var(--border-radius);
      backdrop-filter: blur(10px);
      margin-top: 20px;
    }
    
    .auto-refresh-info {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
      margin-bottom: 15px;
      color: white;
      font-size: 0.95em;
    }
    
    .countdown-badge {
      background: rgba(255,255,255,0.2);
      padding: 4px 12px;
      border-radius: 20px;
      font-weight: 600;
    }
    
    .footer-links {
      display: flex;
      justify-content: center;
      gap: 20px;
      margin-top: 10px;
    }
    
    .footer-link {
      color: rgba(255,255,255,0.8);
      text-decoration: none;
      transition: var(--transition);
    }
    
    .footer-link:hover {
      color: white;
      text-decoration: underline;
    }
    
        /* АДАПТИВНОСТЬ */
    @media (max-width: 1200px) {
      .stats-grid {
        grid-template-columns: repeat(2, 1fr);
      }
    }
    
    @media (max-width: 768px) {
      .container {
        padding: 15px;
      }
      
      .header h1 {
        font-size: 2.2em;
      }
      
      .matches-grid {
        grid-template-columns: 1fr;
      }
      
      .stats-grid {
        grid-template-columns: 1fr;
      }
      
      .controls-grid {
        grid-template-columns: 1fr;
      }
      
      .match-header {
        flex-direction: column;
        gap: 10px;
      }
      
      .match-status {
        align-self: flex-start;
      }
      
      .console-header {
        flex-direction: column;
        gap: 10px;
        align-items: flex-start;
      }
      
      .console-controls {
        align-self: stretch;
        justify-content: space-between;
      }
    }
    
        /* УТИЛИТЫ */
    .no-matches {
      text-align: center;
      padding: 40px 20px;
      background: white;
      border-radius: var(--border-radius);
      box-shadow: var(--box-shadow);
    }
    
    .no-matches-icon {
      font-size: 3em;
      margin-bottom: 15px;
      opacity: 0.5;
    }
    
    .error-message {
      background: var(--danger-color);
      color: white;
      padding: 20px;
      border-radius: var(--border-radius);
      text-align: center;
      margin-bottom: 20px;
      box-shadow: var(--box-shadow);
    }
    
        /* Стили для уведомлений */
    .custom-notification {
      position: fixed;
      top: 20px;
      right: 20px;
      padding: 15px 20px;
      background: #2ecc71;
      color: white;
      border-radius: 8px;
      box-shadow: 0 5px 15px rgba(0,0,0,0.2);
      z-index: 10000;
      max-width: 400px;
      word-wrap: break-word;
      transition: all 0.3s ease;
      transform: translateX(100%);
      opacity: 0;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div class="container">
    <!-- ШАПКА -->
    <div class="header">
      <h1>${CONFIG.TEAM_NAME} Matches</h1>
      <div class="subtitle">Real-time match tracking & monitoring</div>
      <div class="team-badge">🎮 Professional CS2 Team</div>
    </div>
    
    <!-- СООБЩЕНИЕ ОБ ОШИБКЕ -->
   ${matchesError ? `
  <div class="error-message">
      ⚠️ ${matchesError}
    </div>
    ` : ''}
    
    <!-- СЕКЦИЯ МАТЧЕЙ - ТЕПЕРЬ ПЕРВАЯ -->
  <div class="matches-section">
      <h2 class="section-title">🎯 Текущие матчи</h2>
      
  ${matchesInfo.length > 0 ? `
  <div class="matches-grid">
        ${matchesInfo.map((match, index) => `
          <div class="match-card ${match.isLive ? 'live' : 'upcoming'}" 
               data-raw-date="${match.rawDate}" 
               id="match-${index}">
            <div class="match-header">
              <div class="match-teams">
                <div class="team-vs">${match.team1} vs ${match.team2}</div>
              </div>
              <div class="match-status ${match.isLive ? 'status-live' : 'status-upcoming'}">
                ${match.isLive ? '🔴 LIVE' : '⏰ UPCOMING'}
              </div>
            </div>
            
   <div class="match-details">
              <div class="match-tournament">
                <span>🏆</span>
                <span>${match.tournament && match.tournament !== "Турнир" ? match.tournament : 'Турнир'}</span>
              </div>
              <div class="match-time">
                <div class="time-main" id="time-${index}">
                  ${match.timeUntil}
                </div>
                <div class="time-secondary">${match.displayTime}</div>
              </div>
            </div>
            
  <div class="match-meta">
              ${match.bestOf ? `
              <div class="meta-item">
                <span>🎯</span>
                <span>BO${match.bestOf}</span>
              </div>
              ` : ''}
              <div class="meta-item">
                <span>📅</span>
                <span>${new Date(match.rawDate).toLocaleDateString('ru-RU')}</span>
              </div>
              ${match.score ? `
              <div class="meta-item">
                <span>📊</span>
                <span>${match.score}</span>
              </div>
              ` : ''}
            </div>
          </div>
        `).join('')}
      </div>
      ` : `
      <div class="no-matches">
        <div class="no-matches-icon">⏳</div>
        <h3>Матчей не найдено</h3>
        <p>На данный момент активных матчей ${CONFIG.TEAM_NAME} не запланировано</p>
      </div>
      `}
    </div>
    
    <!-- СЕТКА МОНИТОРИНГА -->
   <div class="stats-section">
      <h2 class="section-title">📊 Мониторинг системы</h2>
      
   <div class="stats-grid">
        <!-- Карточка 1: Статус кэша -->
        <div class="stat-card ${status.cacheInfo && status.cacheInfo.cacheAgeMinutes > 30 ? 'error' : status.cacheInfo && status.cacheInfo.cacheAgeMinutes > 15 ? 'warning' : 'success'}" id="stat-cache">
          <h3>🕒 Статус кэша</h3>
          <div class="stat-value">${status.cacheInfo ? status.cacheInfo.cacheAgeMinutes : 'N/A'} мин</div>
          <div class="stat-details">
            <p><strong>Данные:</strong> ${status.cacheInfo ? status.cacheInfo.timestamp : 'N/A'}</p>
            <p><strong>Источник:</strong> ${status.cacheInfo ? status.cacheInfo.source : 'N/A'}</p>
            <p><strong>Матчей:</strong> ${status.cacheInfo ? status.cacheInfo.matchesCount : '0'}</p>
          </div>
        </div>
        
        <!-- Карточка 2: Обновления -->
   <div class="stat-card ${status.timeSinceLastUpdate && status.timeSinceLastUpdate.includes('minutes') && parseInt(status.timeSinceLastUpdate) > 10 ? 'warning' : 'success'}" id="stat-updates">
          <h3>🔄 Обновления</h3>
          <div class="stat-value">${status.timeSinceLastUpdate || 'N/A'}</div>
          <div class="stat-details">
            <p><strong>Успешное:</strong> ${status.lastSuccessfulUpdate || 'N/A'}</p>
            <p><strong>Попытка:</strong> ${status.lastUpdateAttempt || 'N/A'}</p>
            <p><strong>Фоновое:</strong> ${status.lastBackgroundAttempt || 'N/A'}</p>
            <p><strong>Принудительное:</strong> ${status.lastForcedAttempt || 'N/A'}</p>
          </div>
        </div>
        
        <!-- Карточка 3: Ошибки -->
  <div class="stat-card ${status.lastError && status.lastError !== 'нет' ? 'error' : 'success'}" id="stat-errors">
          <h3>🚨 Ошибки</h3>
          <div class="stat-value">${status.errorCount || '0'}</div>
          <div class="stat-details">
            <p><strong>Статус:</strong> ${status.lastError ? 'Есть ошибки' : 'Нет ошибок'}</p>
            <p><strong>В процессе:</strong> ${status.updateInProgress ? '✅ Да' : '❌ Нет'}</p>
          </div>
        </div>
        
        <!-- Карточка 4: API Использование -->
  <div class="stat-card ${status.apiUsage && status.apiUsage.usagePercent > 80 ? 'warning' : 'success'}" id="stat-api">
          <h3>📡 API Использование</h3>
          <div class="stat-value">${status.apiUsage ? status.apiUsage.usagePercent : '0'}%</div>
          <div class="stat-details">
            <p><strong>Сегодня:</strong> ${status.apiUsage ? status.apiUsage.daily : '0'}/${status.apiUsage ? status.apiUsage.dailyLimit : '0'}</p>
            <p><strong>Час:</strong> ${status.apiUsage ? status.apiUsage.hourly : '0'}/${status.apiUsage ? status.apiUsage.hourlyLimit : '0'}</p>
            <div class="api-usage-bar">
              <div class="api-usage-fill" style="width: ${status.apiUsage ? status.apiUsage.usagePercent : 0}%"></div>
            </div>
          </div>
        </div>
        
        <!-- Карточка 5: Интервалы -->
  <div class="stat-card" id="stat-intervals">
          <h3>⚙️ Интервалы</h3>
          <div class="stat-value">${CONFIG.MAX_RETRIES} попытки</div>
          <div class="stat-details">
            <p><strong>Фоновое:</strong> ${status.updateIntervals ? status.updateIntervals.background : 'N/A'}</p>
            <p><strong>По запросу:</strong> ${status.updateIntervals ? status.updateIntervals.userRequest : 'N/A'}</p>
            <p><strong>Минимальный:</strong> ${status.updateIntervals ? status.updateIntervals.minUpdate : 'N/A'}</p>
          </div>
        </div>
        
        <!-- Карточка 6: Производительность -->
   <div class="stat-card" id="stat-performance">
          <h3>🚀 Производительность</h3>
          <div class="stat-value">v3.0</div>
          <div class="stat-details">
            <p><strong>Статус:</strong> ✅ Активен</p>
            <p><strong>Обновлено:</strong> ${new Date().toLocaleDateString('ru-RU')}</p>
            <p><strong>Команда:</strong> ${CONFIG.TEAM_NAME}</p>
          </div>
        </div>
      </div>
      
      <!-- ДЕТАЛИ ОШИБКИ ЕСЛИ ЕСТЬ -->
  ${status.lastError && status.lastError !== 'нет' ? `
  <div class="stat-card error" style="grid-column: 1 / -1;">
        <h3>📋 Детали последней ошибки</h3>
        <div class="error-details">${status.lastErrorDetails || status.lastError}</div>
        <div class="stat-details">
          <p><strong>Время ошибки:</strong> ${status.lastErrorTime || 'N/A'}</p>
        </div>
      </div>
      ` : ''}
    </div>
    
    <!-- КОНСОЛЬ ЛОГОВ -->
  <div class="console-section">
      <h2 class="section-title">📝 Консоль системы</h2>
      
   <div class="console-container">
        <div class="console-header">
          <div class="console-title">Журнал событий системы</div>
          <div class="console-controls">
            <button class="console-btn" onclick="clearSystemLogs()">🗑️ Очистить логи</button>
            <button class="console-btn" onclick="refreshConsole()">🔄 Обновить</button>
          </div>
        </div>
        
   <div class="console-logs" id="consoleLogs">
          ${status.systemLogs && status.systemLogs.length > 0 ? status.systemLogs.map(log => `
            <div class="log-entry">
              <span class="log-time">${log.time}</span>
              <span class="log-level log-level-${log.level}">${log.level.toUpperCase()}</span>
              <span class="log-message">${log.message}</span>
            </div>
          `).join('') : `
            <div class="no-logs">Нет записей в журнале</div>
          `}
        </div>
      </div>
    </div>
    
    <!-- ПАНЕЛЬ УПРАВЛЕНИЯ -->
   <div class="controls-section">
      <h2 class="section-title">🎛️ Управление</h2>
      
  <div class="controls-grid">
        <a href="${scriptUrl}?function=runBackgroundUpdate" class="control-btn btn-success">
          <span>🔄</span>
          <span>Обновить сейчас</span>
        </a>
        <a href="${scriptUrl}?function=fixStaleCache" class="control-btn btn-warning">
          <span>🔧</span>
          <span>Исправить кэш</span>
        </a>
        <a href="${scriptUrl}?function=forceRefresh" class="control-btn btn-warning">
          <span>⚡</span>
          <span>Принудительное обновление</span>
        </a>
        <a href="${scriptUrl}?function=clearCache" class="control-btn btn-danger">
          <span>🗑️</span>
          <span>Очистить кэш</span>
        </a>
        <a href="${scriptUrl}?function=updateStatus" class="control-btn">
          <span>📊</span>
          <span>JSON статус</span>
        </a>
        <a href="${scriptUrl}?function=getNextMatch" class="control-btn">
          <span>🎮</span>
          <span>Следующий матч</span>
        </a>
      </div>
    </div>
    
    <!-- ФУТЕР -->
   <div class="footer">
      <div class="auto-refresh-info">
        <span>🔄 Авто-обновление через:</span>
        <span class="countdown-badge" id="countdown">60</span>
        <span>сек</span>
        <a href="javascript:void(0)" onclick="stopAutoRefresh()" class="footer-link">Остановить</a>
        <span>•</span>
        <a href="javascript:void(0)" onclick="restartAutoRefresh()" class="footer-link">Перезапустить</a>
      </div>
      
  <div class="footer-links">
        <span>📍 Панель управления • Обновлено: ${new Date().toLocaleString('ru-RU')}</span>
        <a href="${scriptUrl}" class="footer-link">Основная страница</a>
      </div>
    </div>
  </div>
  
  <!-- JavaScript -->
  <script>
    let autoRefreshTimer;
    let countdownTimer;
    let countdownSeconds = 60;
    let isRefreshing = false;
    let realTimeInterval;
    let statusRefreshTimer;

    // ФУНКЦИИ РЕАЛЬНОГО ВРЕМЕНИ
    function initializeRealTimeUpdates() {
        console.log('🕒 Инициализация реального времени...');
        
        if (realTimeInterval) {
            clearInterval(realTimeInterval);
        }
        
        function updateMatchTimers() {
            const now = new Date().getTime();
            const matchCards = document.querySelectorAll('.match-card');
            
            matchCards.forEach((card, index) => {
                const timeElement = card.querySelector('.time-main');
                const statusElement = card.querySelector('.match-status');
                const rawDate = card.getAttribute('data-raw-date');
                
                if (!rawDate || !timeElement) return;
                
                const matchTime = parseInt(rawDate);
                const diff = matchTime - now;
                
                if (diff <= 0) {
                    timeElement.textContent = "🔴 ONLINE СЕЙЧАС";
                    timeElement.className = 'time-main live-now';
                    card.classList.add('live');
                    card.classList.remove('upcoming');
                    
                    if (statusElement) {
                        statusElement.textContent = '🔴 LIVE';
                        statusElement.className = 'match-status status-live';
                    }
                } else {
                    const days = Math.floor(diff / (1000 * 60 * 60 * 24));
                    const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
                    const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60));
                    const seconds = Math.floor((diff % (1000 * 60)) / 1000);
                    
                    let displayText = '';
                    if (days > 0) {
                        displayText = \`через \${days}д \${hours}ч \${minutes}м\`;
                    } else if (hours > 0) {
                        displayText = \`через \${hours}ч \${minutes}м \${seconds}с\`;
                    } else if (minutes > 0) {
                        displayText = \`через \${minutes}м \${seconds}с\`;
                    } else {
                        displayText = \`через \${seconds}с\`;
                    }
                    
                    timeElement.textContent = displayText;
                    timeElement.className = 'time-main upcoming-time';
                    
                    if (minutes <= 0 && seconds <= 10) {
                        timeElement.textContent = "🔴 ONLINE СЕЙЧАС";
                        timeElement.className = 'time-main live-now';
                        card.classList.add('live');
                        card.classList.remove('upcoming');
                        
                        if (statusElement) {
                            statusElement.textContent = '🔴 LIVE';
                            statusElement.className = 'match-status status-live';
                        }
                    }
                }
            });
        }
        
        realTimeInterval = setInterval(updateMatchTimers, 1000);
        updateMatchTimers();
        console.log('✅ Таймеры реального времени активированы');
    }
    
    function refreshMatchData() {
        if (isRefreshing) return;
        
        isRefreshing = true;
        console.log('🔄 Обновление данных матчей...');
        
        const url = '${scriptUrl}?function=getMatchesJSON&t=' + new Date().getTime();
        
        fetch(url)
            .then(response => {
                if (!response.ok) throw new Error('HTTP error! status: ' + response.status);
                return response.json();
            })
            .then(data => {
                if (data.hasMatches && data.matches.length > 0) {
                    updateMatchCards(data.matches);
                    console.log('✅ Данные матчей обновлены');
                }
            })
            .catch(error => {
                console.error('❌ Ошибка обновления матчей:', error);
            })
            .finally(() => {
                isRefreshing = false;
            });
    }
    
    function updateMatchCards(matches) {
        const matchesSection = document.querySelector('.matches-section');
        if (!matchesSection) return;
        
        let matchesHTML = '';
        
        if (matches.length > 0) {
            matchesHTML = '<div class="matches-grid">';
            matches.forEach((match, index) => {
                const isLive = match.isLive;
                const isUpcoming = match.isUpcoming;
                
                matchesHTML += \`
                    <div class="match-card \${isLive ? 'live' : 'upcoming'}" 
                         data-raw-date="\${match.rawDate}" 
                         id="match-\${index}">
                        <div class="match-header">
                            <div class="match-teams">
                                <div class="team-vs">\${match.team1} vs \${match.team2}</div>
                            </div>
                            <div class="match-status \${isLive ? 'status-live' : 'status-upcoming'}">
                                \${isLive ? '🔴 LIVE' : '⏰ UPCOMING'}
                            </div>
                        </div>
                        
                        <div class="match-details">
                            <div class="match-tournament">
                                <span>🏆</span>
                                <span>\${match.tournament && match.tournament !== "Турнир" ? match.tournament : 'Турнир'}</span>
                            </div>
                            <div class="match-time">
                                <div class="time-main" id="time-\${index}">
                                    \${match.timeUntil}
                                </div>
                                <div class="time-secondary">\${match.displayTime}</div>
                            </div>
                        </div>
                        
                        <div class="match-meta">
                            \${match.bestOf ? \`
                            <div class="meta-item">
                                <span>🎯</span>
                                <span>BO\${match.bestOf}</span>
                            </div>
                            \` : ''}
                            <div class="meta-item">
                                <span>📅</span>
                                <span>\${new Date(match.rawDate).toLocaleDateString('ru-RU')}</span>
                            </div>
                            \${match.score ? \`
                            <div class="meta-item">
                                <span>📊</span>
                                <span>\${match.score}</span>
                            </div>
                            \` : ''}
                        </div>
                    </div>
                \`;
            });
            matchesHTML += '</div>';
        } else {
            matchesHTML = \`
                <div class="no-matches">
                    <div class="no-matches-icon">⏳</div>
                    <h3>Матчей не найдено</h3>
                    <p>На данный момент активных матчей \${'${CONFIG.TEAM_NAME}'} не запланировано</p>
                </div>
            \`;
        }
        
        const existingGrid = matchesSection.querySelector('.matches-grid');
        const existingNoMatches = matchesSection.querySelector('.no-matches');
        
        if (existingGrid) {
            const tempDiv = document.createElement('div');
            tempDiv.innerHTML = matchesHTML;
            
            if (matches.length > 0) {
                existingGrid.innerHTML = tempDiv.querySelector('.matches-grid').innerHTML;
            } else {
                existingGrid.remove();
                const newNoMatches = tempDiv.querySelector('.no-matches');
                if (newNoMatches) {
                    matchesSection.appendChild(newNoMatches);
                }
            }
        } else if (existingNoMatches) {
            const tempDiv = document.createElement('div');
            tempDiv.innerHTML = matchesHTML;
            
            if (matches.length > 0) {
                existingNoMatches.remove();
                const newGrid = tempDiv.querySelector('.matches-grid');
                if (newGrid) {
                    matchesSection.appendChild(newGrid);
                }
            } else {
                existingNoMatches.outerHTML = matchesHTML;
            }
        } else {
            const tempDiv = document.createElement('div');
            tempDiv.innerHTML = matchesHTML;
            const sectionTitle = matchesSection.querySelector('.section-title');
            if (sectionTitle) {
                sectionTitle.insertAdjacentHTML('afterend', matchesHTML);
            }
        }
        
        initializeRealTimeUpdates();
    }
    
    function refreshSystemStatus() {
        console.log('🔄 Обновление статуса системы...');
        
        const url = '${scriptUrl}?function=getSystemStatus&t=' + new Date().getTime();
        
        fetch(url)
            .then(response => {
                if (!response.ok) throw new Error('HTTP error! status: ' + response.status);
                return response.json();
            })
            .then(status => {
                updateSystemStatus(status);
                console.log('✅ Статус системы обновлен');
            })
            .catch(error => {
                console.error('❌ Ошибка обновления статуса системы:', error);
            });
    }
    
    function updateSystemStatus(status) {
        if (!status) return;
        
        const cacheCard = document.getElementById('stat-cache');
        if (cacheCard && status.cacheInfo) {
            const valueEl = cacheCard.querySelector('.stat-value');
            const detailsEl = cacheCard.querySelector('.stat-details');
            if (valueEl) valueEl.textContent = status.cacheInfo.cacheAgeMinutes + ' мин';
            if (detailsEl) detailsEl.innerHTML = \`
                <p><strong>Данные:</strong> \${status.cacheInfo.timestamp}</p>
                <p><strong>Источник:</strong> \${status.cacheInfo.source}</p>
                <p><strong>Матчей:</strong> \${status.cacheInfo.matchesCount}</p>
            \`;
            const cacheAge = status.cacheInfo.cacheAgeMinutes;
            let cardClass = 'stat-card ';
            if (cacheAge > 30) {
                cardClass += 'error';
            } else if (cacheAge > 15) {
                cardClass += 'warning';
            } else {
                cardClass += 'success';
            }
            cacheCard.className = cardClass;
        }
        
        const updateCard = document.getElementById('stat-updates');
        if (updateCard) {
            const valueEl = updateCard.querySelector('.stat-value');
            const detailsEl = updateCard.querySelector('.stat-details');
            if (valueEl) valueEl.textContent = status.timeSinceLastUpdate || 'N/A';
            if (detailsEl) detailsEl.innerHTML = \`
                <p><strong>Успешное:</strong> \${status.lastSuccessfulUpdate || 'N/A'}</p>
                <p><strong>Попытка:</strong> \${status.lastUpdateAttempt || 'N/A'}</p>
                <p><strong>Фоновое:</strong> \${status.lastBackgroundAttempt || 'N/A'}</p>
                <p><strong>Принудительное:</strong> \${status.lastForcedAttempt || 'N/A'}</p>
            \`;
            const minutesSinceUpdate = status.timeSinceLastUpdate ? parseInt(status.timeSinceLastUpdate) : 999;
            updateCard.className = 'stat-card ' + (minutesSinceUpdate > 10 ? 'warning' : 'success');
        }
        
        const errorsCard = document.getElementById('stat-errors');
        if (errorsCard) {
            const valueEl = errorsCard.querySelector('.stat-value');
            const detailsEl = errorsCard.querySelector('.stat-details');
            if (valueEl) valueEl.textContent = status.errorCount || '0';
            if (detailsEl) detailsEl.innerHTML = \`
                <p><strong>Статус:</strong> \${status.lastError && status.lastError !== 'нет' ? 'Есть ошибки' : 'Нет ошибок'}</p>
                <p><strong>В процессе:</strong> \${status.updateInProgress ? '✅ Да' : '❌ Нет'}</p>
            \`;
            errorsCard.className = 'stat-card ' + (status.lastError && status.lastError !== 'нет' ? 'error' : 'success');
        }
        
        const apiCard = document.getElementById('stat-api');
        if (apiCard && status.apiUsage) {
            const valueEl = apiCard.querySelector('.stat-value');
            const detailsEl = apiCard.querySelector('.stat-details');
            if (valueEl) valueEl.textContent = status.apiUsage.usagePercent + '%';
            if (detailsEl) detailsEl.innerHTML = \`
                <p><strong>Сегодня:</strong> \${status.apiUsage.daily}/\${status.apiUsage.dailyLimit}</p>
                <p><strong>Час:</strong> \${status.apiUsage.hourly}/\${status.apiUsage.hourlyLimit}</p>
                <div class="api-usage-bar">
                    <div class="api-usage-fill" style="width: \${status.apiUsage.usagePercent}%"></div>
                </div>
            \`;
            apiCard.className = 'stat-card ' + (status.apiUsage.usagePercent > 80 ? 'warning' : 'success');
        }
        
        updateConsoleLogs(status.systemLogs);
    }
    
    function updateConsoleLogs(systemLogs) {
        const consoleLogs = document.getElementById('consoleLogs');
        if (!consoleLogs) return;
        
        if (systemLogs && systemLogs.length > 0) {
            consoleLogs.innerHTML = systemLogs.map(log => \`
                <div class="log-entry">
                    <span class="log-time">\${log.time}</span>
                    <span class="log-level log-level-\${log.level}">\${log.level.toUpperCase()}</span>
                    <span class="log-message">\${log.message}</span>
                </div>
            \`).join('');
        } else {
            consoleLogs.innerHTML = '<div class="no-logs">Нет записей в журнале</div>';
        }
    }
    
    function startAutoRefresh() {
        stopAutoRefresh();
        
        countdownSeconds = 60;
        updateCountdown();
        
        countdownTimer = setInterval(() => {
            countdownSeconds--;
            updateCountdown();
            
            if (countdownSeconds <= 0) {
                refreshMatchData();
                countdownSeconds = 60;
            }
        }, 1000);
        
        autoRefreshTimer = setTimeout(() => {
            refreshMatchData();
        }, 60000);
        
        statusRefreshTimer = setInterval(() => {
            refreshSystemStatus();
        }, 30000);
        
        console.log('✅ Авто-обновление запущено');
    }
    
    function refreshConsole() {
        console.log('Обновление консоли...');
        showNotification('🔄 Обновление консоли...', 'info');
        
        const url = '${scriptUrl}?function=statusDashboard&t=' + new Date().getTime();
        
        fetch(url)
            .then(response => response.text())
            .then(html => {
                const parser = new DOMParser();
                const newDoc = parser.parseFromString(html, 'text/html');
                const newConsole = newDoc.querySelector('.console-section');
                const currentConsole = document.querySelector('.console-section');
                
                if (newConsole && currentConsole) {
                    currentConsole.innerHTML = newConsole.innerHTML;
                    reinitializeConsoleHandlers();
                    showNotification('✅ Консоль обновлена', 'success');
                }
            })
            .catch(error => {
                console.error('Ошибка обновления консоли:', error);
                showNotification('❌ Ошибка обновления консоли', 'error');
            });
    }
    
    function clearSystemLogs() {
        if (!confirm('Вы уверены, что хотите очистить системные логи?')) return;
        
        showNotification('🗑️ Очистка логов...', 'info');
        
        const url = '${scriptUrl}?function=clearSystemLogs&t=' + new Date().getTime();
        
        fetch(url)
            .then(response => response.text())
            .then(result => {
                showNotification('✅ Логи очищены', 'success');
                refreshConsole();
            })
            .catch(error => {
                console.error('Ошибка очистки логов:', error);
                showNotification('❌ Ошибка очистки логов', 'error');
            });
    }
    
    function reinitializeConsoleHandlers() {
        const clearBtn = document.querySelector('.console-btn[onclick="clearSystemLogs()"]');
        const refreshBtn = document.querySelector('.console-btn[onclick="refreshConsole()"]');
        
        if (clearBtn) clearBtn.onclick = clearSystemLogs;
        if (refreshBtn) refreshBtn.onclick = refreshConsole;
    }
    
    function reinitializeEventHandlers() {
        console.log('Переинициализация обработчиков...');
        
        document.querySelectorAll('.control-btn').forEach(btn => {
            const newBtn = btn.cloneNode(true);
            btn.parentNode.replaceChild(newBtn, btn);
            
            newBtn.addEventListener('click', function(e) {
                if (this.href && this.href.includes('function=')) {
                    e.preventDefault();
                    handleControlButtonClick(this);
                }
            });
        });
        
        reinitializeConsoleHandlers();
    }
    
    function handleControlButtonClick(button) {
        const originalHTML = button.innerHTML;
        const originalHref = button.href;
        
        button.innerHTML = '<span>⏳</span><span>Загрузка...</span>';
        button.style.opacity = '0.7';
        button.style.pointerEvents = 'none';
        
        const fetchUrl = originalHref + '&t=' + new Date().getTime();
        
        fetch(fetchUrl)
            .then(response => {
                if (!response.ok) throw new Error('HTTP error! status: ' + response.status);
                return response.text();
            })
            .then(result => {
                showNotification(result);
                button.innerHTML = originalHTML;
                button.style.opacity = '1';
                button.style.pointerEvents = 'auto';
                
                if (originalHref.includes('runBackgroundUpdate') || 
                    originalHref.includes('fixStaleCache') || 
                    originalHref.includes('forceRefresh') ||
                    originalHref.includes('clearCache')) {
                    setTimeout(refreshMatchData, 1000);
                }
            })
            .catch(error => {
                console.error('Ошибка действия:', error);
                showNotification('❌ Ошибка: ' + error.message, 'error');
                button.innerHTML = originalHTML;
                button.style.opacity = '1';
                button.style.pointerEvents = 'auto';
            });
    }
    
    function stopAutoRefresh() {
        if (autoRefreshTimer) clearTimeout(autoRefreshTimer);
        if (countdownTimer) clearInterval(countdownTimer);
        if (realTimeInterval) clearInterval(realTimeInterval);
        if (statusRefreshTimer) clearInterval(statusRefreshTimer);
        
        autoRefreshTimer = null;
        countdownTimer = null;
        realTimeInterval = null;
        statusRefreshTimer = null;
        
        updateCountdown('остановлен');
    }
    
    function restartAutoRefresh() {
        stopAutoRefresh();
        startAutoRefresh();
    }
    
    function updateCountdown(text = null) {
        const countdownElement = document.getElementById('countdown');
        if (countdownElement) {
            countdownElement.textContent = text !== null ? text : countdownSeconds;
        }
    }

    function showNotification(message, type = 'info') {
        document.querySelectorAll('.custom-notification').forEach(n => n.remove());
        
        const notification = document.createElement('div');
        notification.className = 'custom-notification';
        notification.style.cssText = \`
            position: fixed;
            top: 20px;
            right: 20px;
            padding: 15px 20px;
            background: \${type === 'error' ? '#e74c3c' : 
                         type === 'warning' ? '#f39c12' : '#2ecc71'};
            color: white;
            border-radius: 8px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.2);
            z-index: 10000;
            max-width: 400px;
            word-wrap: break-word;
            transition: all 0.3s ease;
            transform: translateX(100%);
            opacity: 0;
        \`;
        notification.textContent = message;
        
        document.body.appendChild(notification);
        
        setTimeout(() => {
            notification.style.transform = 'translateX(0)';
            notification.style.opacity = '1';
        }, 100);
        
        setTimeout(() => {
            notification.style.transform = 'translateX(100%)';
            notification.style.opacity = '0';
            setTimeout(() => notification.remove(), 300);
        }, 4000);
    }

    document.addEventListener('DOMContentLoaded', function() {
        console.log('🚀 PARIVISION Matches Dashboard initialized');
        
        initializeRealTimeUpdates();
        startAutoRefresh();
        reinitializeEventHandlers();
        
        setTimeout(refreshSystemStatus, 1000);
        setTimeout(refreshMatchData, 2000);
    });
    
    window.addEventListener('error', function(e) {
        console.error('Global error:', e.error);
    });
    
    window.addEventListener('unhandledrejection', function(e) {
        console.error('Unhandled promise rejection:', e.reason);
        showNotification('⚠️ Произошла ошибка в приложении', 'warning');
        e.preventDefault();
    });
  </script>
</body>
</html>
  `;
  
  return HtmlService.createHtmlOutput(html);
  } catch (error) {
    const errorHtml = `
<!DOCTYPE html>
<html>
<head>
  <title>Ошибка</title>
  <meta charset="utf-8">
  <style>
    body { 
      font-family: 'Segoe UI', Arial, sans-serif; 
      margin: 0; 
      padding: 20px; 
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .error-container { 
      max-width: 500px; 
      background: white; 
      padding: 40px; 
      border-radius: 15px; 
      box-shadow: 0 10px 30px rgba(0,0,0,0.2);
      text-align: center;
    }
    .error-title { 
      color: #e74c3c; 
      font-size: 24px; 
      margin-bottom: 20px; 
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
    }
    .error-message { 
      background: #ffeaa7; 
      padding: 15px; 
      border-radius: 8px; 
      margin: 20px 0; 
      font-family: monospace;
      text-align: left;
    }
    .retry-btn { 
      background: #3498db; 
      color: white; 
      border: none; 
      padding: 12px 24px; 
      border-radius: 8px; 
      cursor: pointer;
      font-size: 16px;
      margin: 10px;
      transition: all 0.3s;
    }
    .retry-btn:hover {
      background: #2980b9;
      transform: translateY(-2px);
    }
  </style>
</head>
<body>
  <div class="error-container">
    <div class="error-title">⚠️ Ошибка загрузки панели управления</div>
    <div class="error-message">
      <strong>${error.toString()}</strong>
    </div>
    <p>Попробуйте обновить страницу или очистить кэш</p>
    <button class="retry-btn" onclick="window.location.reload()">🔄 Обновить страницу</button>
    <p style="margin-top: 20px;">
      <a href="${ScriptApp.getService().getUrl()}?function=clearCache" style="color: #3498db; text-decoration: none;">Очистить кэш</a> | 
      <a href="${ScriptApp.getService().getUrl()}" style="color: #3498db; text-decoration: none;">На главную</a>
    </p>
  </div>
</body>
</html>
    `;
    return HtmlService.createHtmlOutput(errorHtml);
  }
}

    // Функция для отладки
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
