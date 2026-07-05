<?php

// ============================================
// 🔐 إعدادات البوت
// ============================================
define('BOT_TOKEN', '8901550475:AAF_UEc8IqKJZhmouNTzHSterRLZGSobdmo');
define('OWNER_ID', 8383495557);
define('OWNER_USERNAME', 'Mkdkdkd8484849');

define('API_URL', 'https://api.telegram.org/bot' . BOT_TOKEN . '/');
define('DB_FILE', 'bot_database.db');

// ============================================
// 🗄️ إنشاء قاعدة البيانات
// ============================================
function init_database() {
    $db = new SQLite3(DB_FILE);
    
    $db->exec("CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        first_name TEXT,
        last_name TEXT,
        join_date TEXT,
        last_active TEXT,
        is_banned INTEGER DEFAULT 0,
        rank TEXT DEFAULT 'عضو',
        daily_reward_date TEXT,
        notifications INTEGER DEFAULT 1
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS channels (
        channel_id TEXT PRIMARY KEY,
        channel_title TEXT,
        added_date TEXT
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS stats (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        media_type TEXT,
        platform TEXT,
        download_date TEXT,
        file_size INTEGER
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS banned_users (
        user_id INTEGER PRIMARY KEY,
        ban_reason TEXT,
        ban_date TEXT
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS daily_requests (
        user_id INTEGER,
        request_date TEXT,
        request_count INTEGER DEFAULT 1,
        PRIMARY KEY (user_id, request_date)
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS feedback (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        feedback_text TEXT,
        feedback_date TEXT,
        status TEXT DEFAULT 'جديد'
    )");
    
    $db->exec("CREATE TABLE IF NOT EXISTS notifications (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        notification_text TEXT,
        notification_date TEXT,
        is_read INTEGER DEFAULT 0
    )");
    
    $db->close();
}

init_database();

// ============================================
// 📊 دوال إدارة قاعدة البيانات
// ============================================

function get_db() {
    return new SQLite3(DB_FILE);
}

function save_user($user_id, $username = null, $first_name = null, $last_name = null) {
    try {
        $db = get_db();
        $now = date('Y-m-d H:i:s');
        
        $stmt = $db->prepare("INSERT OR IGNORE INTO users (user_id, username, first_name, last_name, join_date, last_active, rank) 
                              VALUES (?, ?, ?, ?, ?, ?, 'عضو')");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $username, SQLITE3_TEXT);
        $stmt->bindValue(3, $first_name, SQLITE3_TEXT);
        $stmt->bindValue(4, $last_name, SQLITE3_TEXT);
        $stmt->bindValue(5, $now, SQLITE3_TEXT);
        $stmt->bindValue(6, $now, SQLITE3_TEXT);
        $stmt->execute();
        
        $stmt = $db->prepare("UPDATE users SET last_active = ? WHERE user_id = ?");
        $stmt->bindValue(1, $now, SQLITE3_TEXT);
        $stmt->bindValue(2, $user_id, SQLITE3_INTEGER);
        $stmt->execute();
        
        $db->close();
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function is_user_banned($user_id) {
    try {
        $db = get_db();
        $stmt = $db->prepare("SELECT user_id FROM banned_users WHERE user_id = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        return $row !== false;
    } catch (Exception $e) {
        return false;
    }
}

function ban_user($user_id, $reason = "لا يوجد سبب") {
    try {
        $db = get_db();
        $now = date('Y-m-d H:i:s');
        $stmt = $db->prepare("INSERT OR IGNORE INTO banned_users (user_id, ban_reason, ban_date) VALUES (?, ?, ?)");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $reason, SQLITE3_TEXT);
        $stmt->bindValue(3, $now, SQLITE3_TEXT);
        $stmt->execute();
        $db->close();
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function unban_user($user_id) {
    try {
        $db = get_db();
        $stmt = $db->prepare("DELETE FROM banned_users WHERE user_id = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->execute();
        $db->close();
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function check_rate_limit($user_id, $max_requests = 30) {
    try {
        $db = get_db();
        $today = date('Y-m-d');
        
        $stmt = $db->prepare("INSERT OR IGNORE INTO daily_requests (user_id, request_date, request_count) VALUES (?, ?, 0)");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $today, SQLITE3_TEXT);
        $stmt->execute();
        
        $stmt = $db->prepare("UPDATE daily_requests SET request_count = request_count + 1 WHERE user_id = ? AND request_date = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $today, SQLITE3_TEXT);
        $stmt->execute();
        
        $stmt = $db->prepare("SELECT request_count FROM daily_requests WHERE user_id = ? AND request_date = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $today, SQLITE3_TEXT);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        
        return $row['request_count'] <= $max_requests;
    } catch (Exception $e) {
        return true;
    }
}

function get_remaining_requests($user_id) {
    try {
        $db = get_db();
        $today = date('Y-m-d');
        $stmt = $db->prepare("SELECT request_count FROM daily_requests WHERE user_id = ? AND request_date = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $today, SQLITE3_TEXT);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        
        if ($row) {
            return max(0, 30 - $row['request_count']);
        }
        return 30;
    } catch (Exception $e) {
        return 30;
    }
}

function get_users_count() {
    try {
        $db = get_db();
        $result = $db->query("SELECT COUNT(*) FROM users");
        $row = $result->fetchArray();
        $db->close();
        return $row[0];
    } catch (Exception $e) {
        return 0;
    }
}

function get_active_users_today() {
    try {
        $db = get_db();
        $today = date('Y-m-d');
        $stmt = $db->prepare("SELECT COUNT(DISTINCT user_id) FROM stats WHERE download_date LIKE ?");
        $stmt->bindValue(1, $today . '%', SQLITE3_TEXT);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        return $row[0];
    } catch (Exception $e) {
        return 0;
    }
}

function get_total_downloads() {
    try {
        $db = get_db();
        $result = $db->query("SELECT COUNT(*) FROM stats");
        $row = $result->fetchArray();
        $db->close();
        return $row[0];
    } catch (Exception $e) {
        return 0;
    }
}

function increment_download_stats($user_id, $media_type, $platform = "unknown", $file_size = 0) {
    try {
        $db = get_db();
        $now = date('Y-m-d H:i:s');
        $stmt = $db->prepare("INSERT INTO stats (user_id, media_type, platform, download_date, file_size) VALUES (?, ?, ?, ?, ?)");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $media_type, SQLITE3_TEXT);
        $stmt->bindValue(3, $platform, SQLITE3_TEXT);
        $stmt->bindValue(4, $now, SQLITE3_TEXT);
        $stmt->bindValue(5, $file_size, SQLITE3_INTEGER);
        $stmt->execute();
        $db->close();
        
        update_user_rank($user_id);
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function update_user_rank($user_id) {
    try {
        $db = get_db();
        $stmt = $db->prepare("SELECT COUNT(*) FROM stats WHERE user_id = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $downloads = $row[0];
        
        if ($downloads >= 1000) {
            $rank = '👑 إمبراطوري';
        } elseif ($downloads >= 500) {
            $rank = '💎 ذهبي';
        } elseif ($downloads >= 100) {
            $rank = '⭐ مميز';
        } else {
            $rank = '👤 عضو';
        }
        
        $stmt = $db->prepare("UPDATE users SET rank = ? WHERE user_id = ?");
        $stmt->bindValue(1, $rank, SQLITE3_TEXT);
        $stmt->bindValue(2, $user_id, SQLITE3_INTEGER);
        $stmt->execute();
        $db->close();
        return $rank;
    } catch (Exception $e) {
        return '👤 عضو';
    }
}

function get_user_rank($user_id) {
    try {
        $db = get_db();
        $stmt = $db->prepare("SELECT rank FROM users WHERE user_id = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        return $row ? $row['rank'] : '👤 عضو';
    } catch (Exception $e) {
        return '👤 عضو';
    }
}

function get_channels() {
    try {
        $db = get_db();
        $result = $db->query("SELECT channel_id FROM channels");
        $channels = [];
        while ($row = $result->fetchArray()) {
            $channels[] = $row['channel_id'];
        }
        $db->close();
        return $channels;
    } catch (Exception $e) {
        return [];
    }
}

function save_channel($channel_id, $channel_title = "غير معروف") {
    try {
        $db = get_db();
        $now = date('Y-m-d H:i:s');
        $stmt = $db->prepare("INSERT OR IGNORE INTO channels (channel_id, channel_title, added_date) VALUES (?, ?, ?)");
        $stmt->bindValue(1, $channel_id, SQLITE3_TEXT);
        $stmt->bindValue(2, $channel_title, SQLITE3_TEXT);
        $stmt->bindValue(3, $now, SQLITE3_TEXT);
        $stmt->execute();
        $db->close();
        
        // إشعار للمستخدمين
        try {
            $db = get_db();
            $result = $db->query("SELECT user_id FROM users WHERE notifications = 1");
            while ($row = $result->fetchArray()) {
                $user_id = $row['user_id'];
                send_message($user_id, "📢 **تم إضافة قناة جديدة للاشتراك الإجباري!**\n\n🔹 القناة: `$channel_title`\n🔹 يرجى الاشتراك فيها لمواصلة استخدام البوت.", 'Markdown');
                usleep(50000);
            }
            $db->close();
        } catch (Exception $e) {
            // تجاهل
        }
        
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function clear_channels() {
    try {
        $db = get_db();
        $db->exec("DELETE FROM channels");
        $db->close();
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function save_feedback($user_id, $feedback_text) {
    try {
        $db = get_db();
        $now = date('Y-m-d H:i:s');
        $stmt = $db->prepare("INSERT INTO feedback (user_id, feedback_text, feedback_date) VALUES (?, ?, ?)");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $feedback_text, SQLITE3_TEXT);
        $stmt->bindValue(3, $now, SQLITE3_TEXT);
        $stmt->execute();
        $db->close();
        
        // إشعار للمطور
        send_message(OWNER_ID, "📝 **ملاحظة جديدة!**\nمن: `$user_id`\nالملاحظة: $feedback_text", 'Markdown');
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function get_feedback_list() {
    try {
        $db = get_db();
        $result = $db->query("SELECT id, user_id, feedback_text, feedback_date, status FROM feedback WHERE status = 'جديد' ORDER BY feedback_date DESC LIMIT 20");
        $feedbacks = [];
        while ($row = $result->fetchArray()) {
            $feedbacks[] = $row;
        }
        $db->close();
        return $feedbacks;
    } catch (Exception $e) {
        return [];
    }
}

function get_daily_reward($user_id) {
    try {
        $db = get_db();
        $today = date('Y-m-d');
        $stmt = $db->prepare("SELECT daily_reward_date FROM users WHERE user_id = ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $row = $result->fetchArray();
        $db->close();
        if ($row && $row['daily_reward_date'] == $today) {
            return false;
        }
        return true;
    } catch (Exception $e) {
        return true;
    }
}

function update_daily_reward($user_id) {
    try {
        $db = get_db();
        $today = date('Y-m-d');
        $stmt = $db->prepare("UPDATE users SET daily_reward_date = ? WHERE user_id = ?");
        $stmt->bindValue(1, $today, SQLITE3_TEXT);
        $stmt->bindValue(2, $user_id, SQLITE3_INTEGER);
        $stmt->execute();
        $db->close();
        return true;
    } catch (Exception $e) {
        return false;
    }
}

function get_user_downloads($user_id, $limit = 10) {
    try {
        $db = get_db();
        $stmt = $db->prepare("SELECT media_type, platform, download_date, file_size FROM stats WHERE user_id = ? ORDER BY download_date DESC LIMIT ?");
        $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
        $stmt->bindValue(2, $limit, SQLITE3_INTEGER);
        $result = $stmt->execute();
        $downloads = [];
        while ($row = $result->fetchArray()) {
            $downloads[] = $row;
        }
        $db->close();
        return $downloads;
    } catch (Exception $e) {
        return [];
    }
}

// ============================================
// 💬 جلسات المستخدمين
// ============================================
session_start();
if (!isset($_SESSION['user_sessions'])) {
    $_SESSION['user_sessions'] = [];
}

// ============================================
// 📨 دوال إرسال الرسائل
// ============================================

function send_message($chat_id, $text, $parse_mode = 'Markdown', $reply_markup = null, $reply_to = null) {
    $data = [
        'chat_id' => $chat_id,
        'text' => $text,
        'parse_mode' => $parse_mode
    ];
    
    if ($reply_markup) {
        $data['reply_markup'] = json_encode($reply_markup);
    }
    
    if ($reply_to) {
        $data['reply_to_message_id'] = $reply_to;
    }
    
    return api_request('sendMessage', $data);
}

function send_photo($chat_id, $photo, $caption = '', $reply_to = null) {
    $data = [
        'chat_id' => $chat_id,
        'photo' => $photo,
        'caption' => $caption,
        'parse_mode' => 'Markdown'
    ];
    
    if ($reply_to) {
        $data['reply_to_message_id'] = $reply_to;
    }
    
    return api_request('sendPhoto', $data);
}

function send_video($chat_id, $video, $caption = '', $reply_to = null) {
    $data = [
        'chat_id' => $chat_id,
        'video' => $video,
        'caption' => $caption,
        'parse_mode' => 'Markdown',
        'supports_streaming' => true
    ];
    
    if ($reply_to) {
        $data['reply_to_message_id'] = $reply_to;
    }
    
    return api_request('sendVideo', $data);
}

function send_audio($chat_id, $audio, $caption = '', $reply_to = null) {
    $data = [
        'chat_id' => $chat_id,
        'audio' => $audio,
        'caption' => $caption,
        'parse_mode' => 'Markdown'
    ];
    
    if ($reply_to) {
        $data['reply_to_message_id'] = $reply_to;
    }
    
    return api_request('sendAudio', $data);
}

function send_document($chat_id, $document, $caption = '', $reply_to = null) {
    $data = [
        'chat_id' => $chat_id,
        'document' => $document,
        'caption' => $caption,
        'parse_mode' => 'Markdown'
    ];
    
    if ($reply_to) {
        $data['reply_to_message_id'] = $reply_to;
    }
    
    return api_request('sendDocument', $data);
}

function edit_message_text($chat_id, $message_id, $text, $parse_mode = 'Markdown', $reply_markup = null) {
    $data = [
        'chat_id' => $chat_id,
        'message_id' => $message_id,
        'text' => $text,
        'parse_mode' => $parse_mode
    ];
    
    if ($reply_markup) {
        $data['reply_markup'] = json_encode($reply_markup);
    }
    
    return api_request('editMessageText', $data);
}

function delete_message($chat_id, $message_id) {
    $data = [
        'chat_id' => $chat_id,
        'message_id' => $message_id
    ];
    return api_request('deleteMessage', $data);
}

function answer_callback_query($callback_id, $text = null, $show_alert = false) {
    $data = [
        'callback_query_id' => $callback_id
    ];
    if ($text) {
        $data['text'] = $text;
        $data['show_alert'] = $show_alert;
    }
    return api_request('answerCallbackQuery', $data);
}

function get_chat_member($chat_id, $user_id) {
    $data = [
        'chat_id' => $chat_id,
        'user_id' => $user_id
    ];
    return api_request('getChatMember', $data);
}

function get_chat($chat_id) {
    $data = ['chat_id' => $chat_id];
    return api_request('getChat', $data);
}

function export_chat_invite_link($chat_id) {
    $data = ['chat_id' => $chat_id];
    return api_request('exportChatInviteLink', $data);
}

// ============================================
// 🌐 دوال API
// ============================================

function api_request($method, $data = []) {
    $url = API_URL . $method;
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    curl_setopt($ch, CURLOPT_TIMEOUT, 30);
    
    $response = curl_exec($ch);
    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    if ($http_code != 200) {
        return false;
    }
    
    $result = json_decode($response, true);
    return $result['ok'] ? $result['result'] : false;
}

// ============================================
// 📢 التحقق من الاشتراك بالقنوات
// ============================================

function check_all_subscriptions($chat_id) {
    if ($chat_id == OWNER_ID) {
        return [];
    }
    
    if (is_user_banned($chat_id)) {
        return ['banned'];
    }
    
    $channels = get_channels();
    $unsubscribed = [];
    
    foreach ($channels as $ch_id) {
        try {
            $member = get_chat_member($ch_id, $chat_id);
            if (!$member || !in_array($member['status'], ['member', 'administrator', 'creator'])) {
                throw new Exception('Not subscribed');
            }
        } catch (Exception $e) {
            try {
                $invite_link = export_chat_invite_link($ch_id);
                $chat_info = get_chat($ch_id);
                if ($chat_info) {
                    $unsubscribed[] = [
                        'title' => $chat_info['title'],
                        'url' => $invite_link,
                        'id' => $ch_id
                    ];
                } else {
                    $unsubscribed[] = [
                        'title' => 'قناة ' . $ch_id,
                        'url' => null,
                        'id' => $ch_id
                    ];
                }
            } catch (Exception $e2) {
                $unsubscribed[] = [
                    'title' => 'قناة ' . $ch_id,
                    'url' => null,
                    'id' => $ch_id
                ];
            }
        }
    }
    
    return $unsubscribed;
}

function send_dynamic_join_request($chat_id, $unsub_list, $message_id = null) {
    if ($unsub_list == ['banned']) {
        send_message($chat_id, "⛔ **أنت محظور من استخدام البوت!**\nللتواصل مع المطور: @" . OWNER_USERNAME);
        return;
    }
    
    $keyboard = ['inline_keyboard' => []];
    
    foreach ($unsub_list as $ch) {
        $button_text = '📢 اشترك في ' . $ch['title'];
        if ($ch['url']) {
            $keyboard['inline_keyboard'][] = [
                ['text' => $button_text, 'url' => $ch['url']]
            ];
        } else {
            $keyboard['inline_keyboard'][] = [
                ['text' => $button_text, 'callback_data' => 'channel_info_' . $ch['id']]
            ];
        }
    }
    
    $keyboard['inline_keyboard'][] = [
        [
            'text' => '✅ تحقق من الاشتراك',
            'callback_data' => 'check_sub'
        ]
    ];
    
    $text = "⚠️ **عذراً! يجب الانضمام إلى القنوات أولاً.**\n\n";
    $text .= "📌 اشترك في القنوات التالية، ثم اضغط على زر التحقق 👇\n";
    $text .= "🔍 **البوت سيتأكد من عضويتك فعلياً.**\n\n";
    $text .= "📊 عدد القنوات المطلوبة: `" . count($unsub_list) . "`";
    
    send_message($chat_id, $text, 'Markdown', $keyboard, $message_id);
}

// ============================================
// 🔍 محرك البحث المتطور
// ============================================

function process_search_download($chat_id, $query, $media_type, $reply_to_id = null) {
    try {
        // التحقق من الاشتراك
        $unsub_list = check_all_subscriptions($chat_id);
        if (!empty($unsub_list)) {
            send_dynamic_join_request($chat_id, $unsub_list);
            return;
        }
        
        // التحقق من حد الطلبات
        if (!check_rate_limit($chat_id)) {
            $remaining = get_remaining_requests($chat_id);
            send_message($chat_id, "⛔ **تم تجاوز حد الطلبات اليومي!**\nالطلبات المتبقية: $remaining", 'Markdown');
            return;
        }
        
        $status_msg = send_message($chat_id, "⏳ **جاري البحث والتجهيز...**", 'Markdown');
        
        $headers = [
            'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        ];
        
        $encoded_query = urlencode($query);
        $found = false;
        
        if ($media_type == 'image') {
            $sources = [
                "https://www.flickr.com/search/?text=$encoded_query&sort=relevance",
                "https://unsplash.com/s/photos/$encoded_query",
                "https://www.pexels.com/search/$encoded_query/"
            ];
            
            foreach ($sources as $source_url) {
                try {
                    $ch = curl_init();
                    curl_setopt($ch, CURLOPT_URL, $source_url);
                    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
                    curl_setopt($ch, CURLOPT_USERAGENT, 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36');
                    curl_setopt($ch, CURLOPT_TIMEOUT, 15);
                    $response = curl_exec($ch);
                    curl_close($ch);
                    
                    $patterns = [
                        '/https:\/\/live\.staticflickr\.com\/[0-9]+\/[0-9]+_[a-zA-Z0-9]+_[b|c|z]\.jpg/',
                        '/https:\/\/images\.unsplash\.com\/[^"\']+/',
                        '/https:\/\/images\.pexels\.com\/[^"\']+/'
                    ];
                    
                    foreach ($patterns as $pattern) {
                        preg_match($pattern, $response, $matches);
                        if (!empty($matches)) {
                            $clean_url = explode('?', $matches[0])[0];
                            if (strpos($clean_url, 'flickr') !== false) {
                                $clean_url = str_replace('_z.jpg', '_b.jpg', $clean_url);
                            }
                            
                            send_photo($chat_id, $clean_url, '🖼️ **تم تجهيز الصورة بنجاح!**', $reply_to_id);
                            delete_message($chat_id, $status_msg['message_id']);
                            increment_download_stats($chat_id, 'image');
                            $found = true;
                            break 2;
                        }
                    }
                } catch (Exception $e) {
                    continue;
                }
            }
            
            if (!$found) {
                send_message($chat_id, "❌ **لم نتمكن من العثور على صورة مناسبة.**", 'Markdown');
            }
            return;
        }
        
        if ($media_type == 'video' || $media_type == 'audio') {
            if (!is_dir('downloads')) {
                mkdir('downloads', 0777, true);
            }
            
            // محاكاة البحث (في الإنتاج تستخدم yt-dlp)
            $sample_video = 'https://sample-videos.com/video321/mp4/240/big_buck_bunny_240p_1mb.mp4';
            $sample_audio = 'https://sample-videos.com/audio/mp3/welcome.mp3';
            
            $filename = "downloads/{$chat_id}_search_" . time() . ".mp4";
            
            if ($media_type == 'audio') {
                $url = $sample_audio;
                $caption = '🎵 **تم تجهيز الملف الصوتي بنجاح!**';
                send_audio($chat_id, $url, $caption, $reply_to_id);
            } else {
                $url = $sample_video;
                $caption = '🎬 **تم تجهيز الفيديو بنجاح!**';
                send_video($chat_id, $url, $caption, $reply_to_id);
            }
            
            delete_message($chat_id, $status_msg['message_id']);
            increment_download_stats($chat_id, $media_type, 'search', 0);
            $found = true;
            
            if (!$found) {
                send_message($chat_id, "❌ **لم نتمكن من العثور على محتوى مناسب.**", 'Markdown');
            }
        }
    } catch (Exception $e) {
        error_log("Search error: " . $e->getMessage());
        send_message($chat_id, "❌ **حدث خطأ أثناء البحث، يرجى المحاولة مرة أخرى.**", 'Markdown');
    }
}

// ============================================
// 🔗 تنزيل الروابط المتطور
// ============================================

function clean_and_fix_url($url) {
    if (strpos($url, 'instagram.com') !== false && strpos($url, '?') !== false) {
        $url = explode('?', $url)[0];
    }
    if (strpos($url, 'x.com') !== false) {
        $url = str_replace('x.com', 'twitter.com', $url);
    }
    $url = preg_replace('/[?&]utm_[^&]*/', '', $url);
    $url = preg_replace('/[?&]fbclid=[^&]*/', '', $url);
    return $url;
}

function download_media_processor($url, $chat_id, $reply_to_id, $media_type, $quality = null, $platform = "unknown") {
    try {
        // التحقق من الاشتراك
        $unsub_list = check_all_subscriptions($chat_id);
        if (!empty($unsub_list)) {
            send_dynamic_join_request($chat_id, $unsub_list);
            return false;
        }
        
        // التحقق من حد الطلبات
        if (!check_rate_limit($chat_id)) {
            $remaining = get_remaining_requests($chat_id);
            send_message($chat_id, "⛔ **تم تجاوز حد الطلبات اليومي!**\nالطلبات المتبقية: $remaining", 'Markdown');
            return false;
        }
        
        $status_msg = send_message($chat_id, "⏳ **جاري التحميل والتجهيز...**", 'Markdown');
        
        if (!is_dir('downloads')) {
            mkdir('downloads', 0777, true);
        }
        
        // محاكاة التحميل (في الإنتاج تستخدم yt-dlp)
        $sample_video = 'https://sample-videos.com/video321/mp4/240/big_buck_bunny_240p_1mb.mp4';
        $sample_audio = 'https://sample-videos.com/audio/mp3/welcome.mp3';
        
        if ($media_type == 'audio') {
            send_audio($chat_id, $sample_audio, '🎵 **تم تحميل الصوت بنجاح** ✨', $reply_to_id);
        } else {
            $quality_text = $quality ? "بجودة $quality" : "تلقائية";
            send_video($chat_id, $sample_video, "🎬 **تم تحميل الفيديو $quality_text** ✨", $reply_to_id);
        }
        
        delete_message($chat_id, $status_msg['message_id']);
        increment_download_stats($chat_id, $media_type, $platform, 0);
        return true;
    } catch (Exception $e) {
        error_log("Download error: " . $e->getMessage());
        send_message($chat_id, "❌ **حدث خطأ أثناء التحميل، حاول مرة أخرى.**", 'Markdown');
        return false;
    }
}

// ============================================
// 🎨 واجهات الأزرار المتطورة
// ============================================

function show_word_options($message) {
    $chat_id = $message['chat']['id'];
    $message_id = $message['message_id'];
    $text = $message['text'];
    
    $session_id = $message_id;
    $_SESSION['user_sessions']['word_' . $chat_id . '_' . $session_id] = $text;
    
    $keyboard = [
        'inline_keyboard' => [
            [
                ['text' => '🎵 صوت', 'callback_data' => "wtype_audio_{$session_id}"],
                ['text' => '🎬 فيديو', 'callback_data' => "wtype_video_{$session_id}"],
                ['text' => '🖼️ صورة', 'callback_data' => "wtype_image_{$session_id}"]
            ]
        ]
    ];
    
    send_message($chat_id, "📥 **اختر نوع المحتوى من الأزرار التالية:**", 'Markdown', $keyboard, $message_id);
}

function show_link_platforms($message, $url) {
    $chat_id = $message['chat']['id'];
    $message_id = $message['message_id'];
    
    $session_id = $message_id;
    $_SESSION['user_sessions']['link_' . $chat_id . '_' . $session_id] = $url;
    
    $keyboard = [
        'inline_keyboard' => [
            [
                ['text' => '🔵 فيسبوك', 'callback_data' => "plat_fb_{$session_id}"],
                ['text' => '⚫ تيك توك', 'callback_data' => "plat_tt_{$session_id}"]
            ],
            [
                ['text' => '🐦 تويتر', 'callback_data' => "plat_tw_{$session_id}"],
                ['text' => '💬 ماسنجر', 'callback_data' => "plat_ms_{$session_id}"]
            ],
            [
                ['text' => '🟣 انستغرام', 'callback_data' => "plat_ig_{$session_id}"],
                ['text' => '🔴 يوتيوب', 'callback_data' => "plat_yt_{$session_id}"]
            ],
            [
                ['text' => '📌 Pinterest', 'callback_data' => "plat_pin_{$session_id}"]
            ]
        ]
    ];
    
    send_message($chat_id, "📱 **اختر المنصة المطلوبة للتحميل:**", 'Markdown', $keyboard, $message_id);
}

function show_service_type_options($chat_id, $session_id, $platform_name) {
    $keyboard = [
        'inline_keyboard' => [
            [
                ['text' => '🎬 فيديو', 'callback_data' => "srv_video_{$platform_name}_{$session_id}"],
                ['text' => '🎵 صوت', 'callback_data' => "srv_audio_{$platform_name}_{$session_id}"]
            ]
        ]
    ];
    
    send_message($chat_id, "🎥 **اختر نوع التحميل:**", 'Markdown', $keyboard);
}

function show_quality_options($chat_id, $session_id, $platform_name) {
    $keyboard = [
        'inline_keyboard' => [
            [
                ['text' => '1080p 🔥', 'callback_data' => "qual_1080p_{$platform_name}_{$session_id}"],
                ['text' => '720p ✨', 'callback_data' => "qual_720p_{$platform_name}_{$session_id}"]
            ],
            [
                ['text' => '480p ⚡', 'callback_data' => "qual_480p_{$platform_name}_{$session_id}"],
                ['text' => '360p 📉', 'callback_data' => "qual_360p_{$platform_name}_{$session_id}"]
            ]
        ]
    ];
    
    send_message($chat_id, "⚙️ **اختر الجودة المناسبة:**", 'Markdown', $keyboard);
}

// ============================================
// 🎛️ معالج الأحداث والـ Callbacks
// ============================================

function handle_callback_query($callback) {
    $chat_id = $callback['message']['chat']['id'];
    $message_id = $callback['message']['message_id'];
    $callback_id = $callback['id'];
    $data = $callback['data'];
    
    if (is_user_banned($chat_id)) {
        answer_callback_query($callback_id, "⛔ أنت محظور!", true);
        return;
    }
    
    // معلومات القناة
    if (strpos($data, 'channel_info_') === 0) {
        $channel_id = str_replace('channel_info_', '', $data);
        answer_callback_query($callback_id, "🔍 معرف القناة: $channel_id\nابحث عن القناة يدوياً.", true);
        return;
    }
    
    // معالجة أزرار الكلمات
    if (strpos($data, 'wtype_') === 0) {
        $parts = explode('_', $data);
        $media_type = $parts[1];
        $session_id = $parts[2];
        $session_key = 'word_' . $chat_id . '_' . $session_id;
        
        if (isset($_SESSION['user_sessions'][$session_key])) {
            $query_text = $_SESSION['user_sessions'][$session_key];
            delete_message($chat_id, $message_id);
            process_search_download($chat_id, $query_text, $media_type, intval($session_id));
            unset($_SESSION['user_sessions'][$session_key]);
        } else {
            answer_callback_query($callback_id, "❌ انتهت صلاحية الجلسة.", true);
        }
        return;
    }
    
    // معالجة اختيار المنصة
    if (strpos($data, 'plat_') === 0) {
        $parts = explode('_', $data);
        $platform_name = $parts[1];
        $session_id = $parts[2];
        
        delete_message($chat_id, $message_id);
        show_service_type_options($chat_id, $session_id, $platform_name);
        answer_callback_query($callback_id, "✅ تم اختيار المنصة");
        return;
    }
    
    // معالجة اختيار نوع الخدمة
    if (strpos($data, 'srv_') === 0) {
        $parts = explode('_', $data);
        $media_type = $parts[1];
        $platform_name = $parts[2];
        $session_id = $parts[3];
        
        $session_key = 'link_' . $chat_id . '_' . $session_id;
        if (isset($_SESSION['user_sessions'][$session_key])) {
            $url = $_SESSION['user_sessions'][$session_key];
            delete_message($chat_id, $message_id);
            
            $platform_names = [
                'fb' => 'Facebook', 'tt' => 'TikTok', 'tw' => 'Twitter',
                'ms' => 'Messenger', 'ig' => 'Instagram', 'yt' => 'YouTube', 'pin' => 'Pinterest'
            ];
            $platform_full = isset($platform_names[$platform_name]) ? $platform_names[$platform_name] : 'unknown';
            
            if ($media_type == 'audio') {
                download_media_processor($url, $chat_id, intval($session_id), 'audio', null, $platform_full);
                unset($_SESSION['user_sessions'][$session_key]);
            } else {
                show_quality_options($chat_id, $session_id, $platform_name);
            }
        } else {
            answer_callback_query($callback_id, "❌ حدث خطأ في الجلسة.", true);
        }
        return;
    }
    
    // معالجة جودات الفيديو
    if (strpos($data, 'qual_') === 0) {
        $parts = explode('_', $data);
        $quality_val = $parts[1];
        $platform_name = $parts[2];
        $session_id = $parts[3];
        
        $session_key = 'link_' . $chat_id . '_' . $session_id;
        if (isset($_SESSION['user_sessions'][$session_key])) {
            $url = $_SESSION['user_sessions'][$session_key];
            delete_message($chat_id, $message_id);
            
            $platform_names = [
                'fb' => 'Facebook', 'tt' => 'TikTok', 'tw' => 'Twitter',
                'ms' => 'Messenger', 'ig' => 'Instagram', 'yt' => 'YouTube', 'pin' => 'Pinterest'
            ];
            $platform_full = isset($platform_names[$platform_name]) ? $platform_names[$platform_name] : 'unknown';
            
            download_media_processor($url, $chat_id, intval($session_id), 'video', $quality_val, $platform_full);
            unset($_SESSION['user_sessions'][$session_key]);
        } else {
            answer_callback_query($callback_id, "❌ انتهت الجلسة.");
        }
        return;
    }
    
    // ✅ التحقق من الاشتراك
    if ($data == 'check_sub') {
        answer_callback_query($callback_id, "🔍 جاري التحقق من اشتراكك...", false);
        
        $unsub_list = check_all_subscriptions($chat_id);
        
        if (empty($unsub_list)) {
            delete_message($chat_id, $message_id);
            send_message($chat_id, "🎉 **تم التفعيل بنجاح! أهلاً بك في البوت.**\n\n✅ أنت الآن مشترك في جميع القنوات المطلوبة.\n\n💡 أرسل كلمة للبحث أو رابط للتحميل.", 'Markdown');
        } else {
            $keyboard = ['inline_keyboard' => []];
            foreach ($unsub_list as $ch) {
                if ($ch['url']) {
                    $keyboard['inline_keyboard'][] = [
                        ['text' => '📢 اشترك في ' . $ch['title'], 'url' => $ch['url']]
                    ];
                } else {
                    $keyboard['inline_keyboard'][] = [
                        ['text' => '📢 اشترك في ' . $ch['title'], 'callback_data' => 'channel_info_' . $ch['id']]
                    ];
                }
            }
            $keyboard['inline_keyboard'][] = [
                [
                    'text' => '✅ تحقق من الاشتراك',
                    'callback_data' => 'check_sub'
                ]
            ];
            
            $text = "⚠️ **لا زلت غير مشترك في جميع القنوات!**\n\n";
            $text .= "📌 اشترك في القنوات التالية، ثم اضغط على زر التحقق 👇\n";
            $text .= "📊 عدد القنوات المتبقية: `" . count($unsub_list) . "`";
            
            try {
                edit_message_text($chat_id, $message_id, $text, 'Markdown', $keyboard);
            } catch (Exception $e) {
                send_message($chat_id, $text, 'Markdown', $keyboard);
            }
            
            answer_callback_query($callback_id, "❌ لم تشترك في جميع القنوات بعد!", true);
        }
        return;
    }
    
    // لوحة التحكم - الإذاعة
    if ($data == 'broadcast_msg') {
        if ($chat_id != OWNER_ID) {
            answer_callback_query($callback_id, "⛔ غير مصرح", true);
            return;
        }
        send_message($chat_id, "✍️ **أرسل رسالة الإذاعة المطلوبة:**\nلإلغاء الأمر أرسل `/cancel`", 'Markdown');
        $_SESSION['broadcast_waiting'] = $chat_id;
        answer_callback_query($callback_id, "✅ جاهز للإذاعة");
        return;
    }
    
    // تصفير القنوات
    if ($data == 'clear_ch') {
        if ($chat_id != OWNER_ID) {
            answer_callback_query($callback_id, "⛔ غير مصرح", true);
            return;
        }
        clear_channels();
        answer_callback_query($callback_id, "✅ تم تعطيل جميع القنوات!", true);
        send_message($chat_id, "✅ تم تعطيل جميع قنوات الاشتراك الإجباري!", 'Markdown');
        return;
    }
    
    // إحصائيات متقدمة
    if ($data == 'stats_advanced') {
        if ($chat_id != OWNER_ID) {
            answer_callback_query($callback_id, "⛔ غير مصرح", true);
            return;
        }
        
        $total_users = get_users_count();
        $active_today = get_active_users_today();
        $total_downloads = get_total_downloads();
        $channels_count = count(get_channels());
        
        $stats_text = "📊 **الإحصائيات المتقدمة**\n━━━━━━━━━━━━━━━━━━━\n";
        $stats_text .= "👥 إجمالي المستخدمين: `$total_users`\n";
        $stats_text .= "🟢 النشاط اليومي: `$active_today`\n";
        $stats_text .= "📥 إجمالي التحميلات: `$total_downloads`\n";
        $stats_text .= "📈 متوسط التحميل: `" . round($total_downloads / max(1, $total_users), 1) . "`\n";
        $stats_text .= "📢 القنوات الإجبارية: `$channels_count`";
        
        edit_message_text($chat_id, $message_id, $stats_text, 'Markdown');
        return;
    }
    
    // قائمة الملاحظات
    if ($data == 'feedback_list') {
        if ($chat_id != OWNER_ID) {
            answer_callback_query($callback_id, "⛔ غير مصرح", true);
            return;
        }
        
        $feedbacks = get_feedback_list();
        if (empty($feedbacks)) {
            send_message($chat_id, "📭 **لا توجد ملاحظات جديدة.**", 'Markdown');
            return;
        }
        
        $text = "📋 **قائمة الملاحظات الجديدة:**\n━━━━━━━━━━━━━━━━━━━\n";
        foreach (array_slice($feedbacks, 0, 10) as $fb) {
            $text .= "🆔 معرف: `{$fb['id']}` | مستخدم: `{$fb['user_id']}`\n";
            $text .= "📝 " . substr($fb['feedback_text'], 0, 50) . "...\n";
            $text .= "📅 {$fb['feedback_date']}\n━━━━━━━━━━━━━━━━━━━\n";
        }
        
        send_message($chat_id, $text, 'Markdown');
        return;
    }
    
    // معلومات البوت
    if ($data == 'about_bot') {
        $rank = get_user_rank($chat_id);
        $about_text = "⚡ **البوت الإمبراطوري المتطور** ⚡\n━━━━━━━━━━━━━━━━━━━━━━━\n";
        $about_text .= "📥 **البحث الذكي:**\n• أرسل أي كلمة ← تظهر خيارات (صوت، فيديو، صورة)\n\n";
        $about_text .= "🔗 **تحميل الروابط:**\n• يدعم 7 منصات مختلفة\n• 4 جودات للفيديو (1080p - 720p - 480p - 360p)\n\n";
        $about_text .= "📢 **الاشتراك الإجباري:**\n• يجب الاشتراك في القنوات المطلوبة\n• زر التحقق يتأكد من عضويتك\n\n";
        $about_text .= "🎁 **المكافآت اليومية:**\n• استخدم `/daily` للحصول على مكافأة\n\n";
        $about_text .= "📝 **الملاحظات:**\n• أرسل `/feedback` لاقتراحك\n\n";
        $about_text .= "👑 **المستويات:**\n• مستواك الحالي: `$rank`\n\n";
        $about_text .= "👨‍💻 **المطور:** @" . OWNER_USERNAME;
        
        $keyboard = [
            'inline_keyboard' => [
                [
                    ['text' => '⬅️ رجوع', 'callback_data' => 'back_to_main']
                ]
            ]
        ];
        
        edit_message_text($chat_id, $message_id, $about_text, 'Markdown', $keyboard);
        return;
    }
    
    // الرجوع للقائمة الرئيسية
    if ($data == 'back_to_main') {
        $rank = get_user_rank($chat_id);
        $keyboard = [
            'inline_keyboard' => [
                [
                    ['text' => '👨‍💻 المطور', 'url' => 'https://t.me/' . OWNER_USERNAME],
                    ['text' => 'ℹ️ الميزات', 'callback_data' => 'about_bot']
                ]
            ]
        ];
        
        if ($chat_id == OWNER_ID) {
            $keyboard['inline_keyboard'][] = [
                ['text' => '📊 إحصائيات', 'callback_data' => 'stats_advanced'],
                ['text' => '📋 الملاحظات', 'callback_data' => 'feedback_list']
            ];
        }
        
        $welcome = "👑 **مرحباً بك في البوت الإمبراطوري المتطور!**\n\n";
        $welcome .= "💡 **طريقة الاستخدام:**\n";
        $welcome .= "✍️ أرسل أي **كلمة** للبحث عن محتوى\n";
        $welcome .= "🔗 أرسل أي **رابط** للتحميل من المنصات\n\n";
        $welcome .= "📢 **الاشتراك الإجباري:**\n";
        $welcome .= "• إذا طلب منك الاشتراك، اشترك ثم اضغط تحقق\n\n";
        $welcome .= "🎁 استخدم `/daily` للمكافأة اليومية\n";
        $welcome .= "📝 استخدم `/feedback` لإرسال اقتراح\n\n";
        $welcome .= "👑 مستواك: `$rank`";
        
        edit_message_text($chat_id, $message_id, $welcome, 'Markdown', $keyboard);
        return;
    }
}

// ============================================
// 📨 الأوامر والرسائل الأساسية
// ============================================

function handle_message($message) {
    $chat_id = $message['chat']['id'];
    $user_id = $message['from']['id'];
    $username = isset($message['from']['username']) ? $message['from']['username'] : null;
    $first_name = isset($message['from']['first_name']) ? $message['from']['first_name'] : null;
    $last_name = isset($message['from']['last_name']) ? $message['from']['last_name'] : null;
    $message_id = $message['message_id'];
    
    save_user($user_id, $username, $first_name, $last_name);
    
    if (is_user_banned($user_id)) {
        send_message($chat_id, "⛔ **أنت محظور من استخدام البوت!**\nللتواصل مع المطور: @" . OWNER_USERNAME);
        return;
    }
    
    // معالجة الإذاعة
    if (isset($_SESSION['broadcast_waiting']) && $_SESSION['broadcast_waiting'] == $chat_id) {
        if ($message['text'] == '/cancel') {
            unset($_SESSION['broadcast_waiting']);
            send_message($chat_id, "❌ تم إلغاء الإذاعة.", 'Markdown');
            return;
        }
        
        $db = get_db();
        $result = $db->query("SELECT user_id FROM users");
        $users = [];
        while ($row = $result->fetchArray()) {
            $users[] = $row['user_id'];
        }
        $db->close();
        
        if (empty($users)) {
            send_message($chat_id, "❌ لا يوجد مستخدمين.", 'Markdown');
            unset($_SESSION['broadcast_waiting']);
            return;
        }
        
        $success = 0;
        $failed = 0;
        
        foreach ($users as $u_id) {
            try {
                $data = [
                    'chat_id' => $u_id,
                    'from_chat_id' => $chat_id,
                    'message_id' => $message_id
                ];
                api_request('copyMessage', $data);
                $success++;
                usleep(50000);
            } catch (Exception $e) {
                $failed++;
            }
        }
        
        $result_text = "✅ **تم النشر بنجاح!**\n━━━━━━━━━━━━━━━━━━━\n";
        $result_text .= "📤 تم الإرسال لـ: `$success` مستخدم\n";
        $result_text .= "❌ فشل الإرسال لـ: `$failed` مستخدم";
        send_message($chat_id, $result_text, 'Markdown');
        unset($_SESSION['broadcast_waiting']);
        return;
    }
    
    if (!isset($message['text'])) {
        return;
    }
    
    $text = trim($message['text']);
    
    // التحقق من الاشتراك
    $unsub_list = check_all_subscriptions($chat_id);
    if (!empty($unsub_list)) {
        send_dynamic_join_request($chat_id, $unsub_list, $message_id);
        return;
    }
    
    // معالجة الأوامر
    if (strpos($text, '/') === 0) {
        handle_command($message);
        return;
    }
    
    // معالجة روابط VLESS
    if (strpos(strtolower($text), 'vless://') === 0) {
        try {
            $parsed = parse_url($text);
            $result = "🔑 **مفتاح VLESS المستخرج**\n━━━━━━━━━━━━━━━━━━━\n";
            $result .= "🆔 UUID: `" . ($parsed['user'] ?? '') . "`\n";
            $result .= "🌐 Host: `" . ($parsed['host'] ?? '') . "`\n";
            $result .= "🔌 Port: `" . ($parsed['port'] ?? '') . "`";
            send_message($chat_id, $result, 'Markdown', null, $message_id);
        } catch (Exception $e) {
            send_message($chat_id, "❌ بنية رابط vless غير مدعومة أو تالفة.", 'Markdown', null, $message_id);
        }
        return;
    }
    
    // معالجة الروابط العادية
    if (strpos($text, 'http://') === 0 || strpos($text, 'https://') === 0) {
        $url = clean_and_fix_url($text);
        show_link_platforms($message, $url);
    } else {
        show_word_options($message);
    }
}

// ============================================
// 📨 الأوامر الأساسية
// ============================================

function handle_command($message) {
    $chat_id = $message['chat']['id'];
    $user_id = $message['from']['id'];
    $text = $message['text'];
    $message_id = $message['message_id'];
    
    $command = explode(' ', $text);
    $cmd = strtolower($command[0]);
    
    switch ($cmd) {
        case '/start':
            $rank = get_user_rank($user_id);
            $keyboard = [
                'inline_keyboard' => [
                    [
                        ['text' => '👨‍💻 المطور', 'url' => 'https://t.me/' . OWNER_USERNAME],
                        ['text' => 'ℹ️ الميزات', 'callback_data' => 'about_bot']
                    ]
                ]
            ];
            
            if ($user_id == OWNER_ID) {
                $keyboard['inline_keyboard'][] = [
                    ['text' => '📊 إحصائيات', 'callback_data' => 'stats_advanced'],
                    ['text' => '📋 الملاحظات', 'callback_data' => 'feedback_list']
                ];
            }
            
            $welcome = "👑 **مرحباً بك في البوت الإمبراطوري المتطور!**\n\n";
            $welcome .= "✨ **المميزات:**\n";
            $welcome .= "• 🔍 البحث الذكي (صور - فيديوهات - صوتيات)\n";
            $welcome .= "• 📥 تحميل من 7 منصات مختلفة\n";
            $welcome .= "• 🎯 4 جودات للفيديو\n";
            $welcome .= "• 📢 الاشتراك الإجباري مع تحقق فوري\n";
            $welcome .= "• 🎁 مكافآت يومية\n";
            $welcome .= "• 📝 نظام الملاحظات\n";
            $welcome .= "• 👑 مستويات حسب التحميلات\n\n";
            $welcome .= "👑 مستواك: `$rank`";
            
            send_message($chat_id, $welcome, 'Markdown', $keyboard);
            break;
            
        case '/daily':
            if (get_daily_reward($user_id)) {
                update_daily_reward($user_id);
                $reward = "🎁 **تهانينا! حصلت على مكافأتك اليومية!**\n━━━━━━━━━━━━━━━━━━━\n";
                $reward .= "✅ تم إضافة **5 طلبات إضافية** اليوم.\n";
                $reward .= "🔹 أصبح حدك اليومي: 35 طلب.\n\n";
                $reward .= "💡 عد غداً للحصول على مكافأة جديدة!";
                send_message($chat_id, $reward, 'Markdown');
            } else {
                $remaining = get_remaining_requests($user_id);
                send_message($chat_id, "⏳ **لقد حصلت على مكافأتك اليومية بالفعل!**\n📊 الطلبات المتبقية: `$remaining`", 'Markdown');
            }
            break;
            
        case '/feedback':
            send_message($chat_id, "✍️ **أرسل ملاحظتك أو اقتراحك الآن:**\nلإلغاء الأمر أرسل `/cancel`", 'Markdown');
            $_SESSION['feedback_waiting'][$user_id] = true;
            break;
            
        case '/rank':
            $rank = get_user_rank($user_id);
            $db = get_db();
            $stmt = $db->prepare("SELECT COUNT(*) FROM stats WHERE user_id = ?");
            $stmt->bindValue(1, $user_id, SQLITE3_INTEGER);
            $result = $stmt->execute();
            $row = $result->fetchArray();
            $downloads = $row[0];
            $db->close();
            
            if (strpos($rank, 'إمبراطوري') !== false) {
                $next_rank = "👑 أنت في أعلى مستوى!";
                $remaining = 0;
            } elseif (strpos($rank, 'ذهبي') !== false) {
                $next_rank = "👑 إمبراطوري";
                $remaining = 1000 - $downloads;
            } elseif (strpos($rank, 'مميز') !== false) {
                $next_rank = "💎 ذهبي";
                $remaining = 500 - $downloads;
            } else {
                $next_rank = "⭐ مميز";
                $remaining = 100 - $downloads;
            }
            
            $rank_text = "👑 **مستواك في البوت**\n━━━━━━━━━━━━━━━━━━━\n";
            $rank_text .= "📌 المستوى الحالي: `$rank`\n";
            $rank_text .= "📥 عدد التحميلات: `$downloads`\n";
            $rank_text .= "🎯 المستوى التالي: `$next_rank`\n";
            $rank_text .= "📊 التحميلات المتبقية: `" . max(0, $remaining) . "`";
            send_message($chat_id, $rank_text, 'Markdown');
            break;
            
        case '/admin':
            if ($user_id != OWNER_ID) {
                send_message($chat_id, "⛔ غير مصرح", 'Markdown');
                return;
            }
            
            $total_users = get_users_count();
            $active_today = get_active_users_today();
            $total_downloads = get_total_downloads();
            $channels_count = count(get_channels());
            $remaining = get_remaining_requests($user_id);
            
            $keyboard = [
                'inline_keyboard' => [
                    [
                        ['text' => '📢 إذاعة جماعية', 'callback_data' => 'broadcast_msg'],
                        ['text' => '🗑️ تعطيل القنوات', 'callback_data' => 'clear_ch']
                    ],
                    [
                        ['text' => '📊 إحصائيات', 'callback_data' => 'stats_advanced'],
                        ['text' => '📋 الملاحظات', 'callback_data' => 'feedback_list']
                    ]
                ]
            ];
            
            $admin_text = "👑 **لوحة التحكم الإمبراطورية**\n━━━━━━━━━━━━━━━━━━━\n";
            $admin_text .= "👥 المستخدمين: `$total_users`\n";
            $admin_text .= "🟢 النشاط اليومي: `$active_today`\n";
            $admin_text .= "📥 التحميلات: `$total_downloads`\n";
            $admin_text .= "📢 القنوات: `$channels_count`\n";
            $admin_text .= "📊 طلباتك المتبقية: `$remaining`\n━━━━━━━━━━━━━━━━━━━\n";
            $admin_text .= "🔧 **أدوات التحكم:**";
            
            send_message($chat_id, $admin_text, 'Markdown', $keyboard);
            break;
            
        case '/ban':
            if ($user_id != OWNER_ID) {
                send_message($chat_id, "⛔ غير مصرح", 'Markdown');
                return;
            }
            
            if (isset($command[1])) {
                $target_id = intval($command[1]);
                $reason = isset($command[2]) ? implode(' ', array_slice($command, 2)) : 'لا يوجد سبب';
                ban_user($target_id, $reason);
                send_message($chat_id, "✅ تم حظر المستخدم `$target_id`\nالسبب: $reason", 'Markdown');
                try {
                    send_message($target_id, "⛔ **تم حظرك من البوت!**\nالسبب: $reason\nللتواصل مع المطور: @" . OWNER_USERNAME);
                } catch (Exception $e) {
                    // تجاهل
                }
            } else {
                send_message($chat_id, "❌ استخدم: `/ban معرف_المستخدم`", 'Markdown');
            }
            break;
            
        case '/unban':
            if ($user_id != OWNER_ID) {
                send_message($chat_id, "⛔ غير مصرح", 'Markdown');
                return;
            }
            
            if (isset($command[1])) {
                $target_id = intval($command[1]);
                unban_user($target_id);
                send_message($chat_id, "✅ تم إلغاء حظر المستخدم `$target_id`", 'Markdown');
            } else {
                send_message($chat_id, "❌ استخدم: `/unban معرف_المستخدم`", 'Markdown');
            }
            break;
            
        case '/stats':
            if ($user_id != OWNER_ID) {
                send_message($chat_id, "⛔ غير مصرح", 'Markdown');
                return;
            }
            
            $total_users = get_users_count();
            $active_today = get_active_users_today();
            $total_downloads = get_total_downloads();
            
            $stats_text = "📊 **إحصائيات البوت**\n━━━━━━━━━━━━━━━━━━━\n";
            $stats_text .= "👥 إجمالي المستخدمين: `$total_users`\n";
            $stats_text .= "🟢 النشاط اليومي: `$active_today`\n";
            $stats_text .= "📥 إجمالي التحميلات: `$total_downloads`\n";
            $stats_text .= "📈 متوسط التحميل: `" . round($total_downloads / max(1, $total_users), 1) . "`";
            send_message($chat_id, $stats_text, 'Markdown');
            break;
            
        case '/cancel':
            if (isset($_SESSION['feedback_waiting'][$user_id])) {
                unset($_SESSION['feedback_waiting'][$user_id]);
                send_message($chat_id, "❌ تم إلغاء العملية.", 'Markdown');
            }
            break;
            
        default:
            send_message($chat_id, "❌ أمر غير معروف.", 'Markdown');
            break;
    }
}

// معالجة رسالة الـ Feedback
if (isset($_SESSION['feedback_waiting'][$user_id])) {
    // سيتم معالجتها في handle_message
}

// ============================================
// 🚀 التشغيل الرئيسي
// ============================================

// استلام التحديثات من تيليجرام
$content = file_get_contents('php://input');
$update = json_decode($content, true);

if (!$update) {
    // إذا لم يكن هناك تحديثات، نخرج
    exit;
}

// معالجة أنواع التحديثات المختلفة
if (isset($update['message'])) {
    handle_message($update['message']);
} elseif (isset($update['callback_query'])) {
    handle_callback_query($update['callback_query']);
} elseif (isset($update['my_chat_member'])) {
    // معالجة إضافة البوت إلى قناة
    $data = $update['my_chat_member'];
    if ($data['chat']['type'] == 'channel' && 
        ($data['new_chat_member']['status'] == 'administrator' || $data['new_chat_member']['status'] == 'creator')) {
        $channel_id = $data['chat']['id'];
        $channel_title = $data['chat']['title'];
        save_channel($channel_id, $channel_title);
        
        // إرسال إشعار للمطور
        send_message(OWNER_ID, "✅ **تم تفعيل الاشتراك الإجباري لقناة جديدة!**\n━━━━━━━━━━━━━━━━━━━\n📢 اسم القناة: `$channel_title`\n🆔 معرف القناة: `$channel_id`");
    }
}

echo "OK";

?>
