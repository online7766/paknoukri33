<?php
/**
 * Website Daily Post Counter
 * Version: 1.0
 * Author: Your Name
 */

// Error reporting
error_reporting(E_ALL);
ini_set('display_errors', 1);

// Set timezone for Pakistan
date_default_timezone_set('Asia/Karachi');

class WebsitePostCounter {
    private $today;
    private $website_url;
    private $domain;
    private $post_count;
    
    public function __construct($url) {
        $this->today = date('Y-m-d');
        $this->website_url = $this->normalizeUrl($url);
        $this->domain = parse_url($this->website_url, PHP_URL_HOST);
        $this->post_count = 0;
    }
    
    private function normalizeUrl($url) {
        if (!preg_match("~^(?:f|ht)tps?://~i", $url)) {
            $url = "https://" . $url;
        }
        return rtrim($url, '/');
    }
    
    public function checkTodayPosts() {
        echo "<h2>🔍 Checking: {$this->website_url}</h2>";
        echo "<p>📅 Today's Date: {$this->today}</p>";
        
        // Try different methods
        $this->tryRSSFeed();
        $this->trySitemap();
        $this->tryHomepage();
        
        return $this->post_count;
    }
    
    private function tryRSSFeed() {
        $feed_urls = [
            '/feed',
            '/rss',
            '/feed.xml',
            '/rss.xml',
            '/atom.xml'
        ];
        
        foreach ($feed_urls as $feed_path) {
            $feed_url = $this->website_url . $feed_path;
            if ($this->checkFeed($feed_url)) {
                break;
            }
        }
    }
    
    private function checkFeed($feed_url) {
        echo "<p>🔗 Trying RSS feed: {$feed_url}</p>";
        
        $content = @file_get_contents($feed_url);
        if (!$content) {
            echo "<p style='color: orange;'>⚠️ Feed not found</p>";
            return false;
        }
        
        echo "<p style='color: green;'>✅ RSS Feed found!</p>";
        
        // Simple XML parsing for RSS/Atom
        try {
            $xml = simplexml_load_string($content);
            
            if ($xml) {
                $today_posts = 0;
                
                // Check for RSS format
                if (isset($xml->channel->item)) {
                    foreach ($xml->channel->item as $item) {
                        if ($this->isToday($item->pubDate)) {
                            $today_posts++;
                        }
                    }
                }
                // Check for Atom format
                elseif (isset($xml->entry)) {
                    foreach ($xml->entry as $entry) {
                        if ($this->isToday($entry->published)) {
                            $today_posts++;
                        }
                    }
                }
                
                if ($today_posts > 0) {
                    $this->post_count = $today_posts;
                    echo "<p style='color: blue;'>📊 Found {$today_posts} posts from today in RSS feed</p>";
                    return true;
                }
            }
        } catch (Exception $e) {
            echo "<p style='color: red;'>❌ Error parsing feed: " . $e->getMessage() . "</p>";
        }
        
        return false;
    }
    
    private function trySitemap() {
        if ($this->post_count > 0) return;
        
        $sitemap_url = $this->website_url . '/sitemap.xml';
        echo "<p>🔗 Trying sitemap: {$sitemap_url}</p>";
        
        $content = @file_get_contents($sitemap_url);
        if (!$content) {
            echo "<p style='color: orange;'>⚠️ Sitemap not found</p>";
            return;
        }
        
        echo "<p style='color: green;'>✅ Sitemap found!</p>";
        
        // Parse sitemap (basic parsing)
        preg_match_all('/<loc>(.*?)<\/loc>/', $content, $urls);
        preg_match_all('/<lastmod>(.*?)<\/lastmod>/', $content, $dates);
        
        $today_count = 0;
        if (!empty($urls[1])) {
            for ($i = 0; $i < count($urls[1]); $i++) {
                $date = isset($dates[1][$i]) ? $dates[1][$i] : '';
                if (strpos($date, $this->today) !== false) {
                    $today_count++;
                }
            }
            
            if ($today_count > 0) {
                $this->post_count = $today_count;
                echo "<p style='color: blue;'>📊 Found {$today_count} posts from today in sitemap</p>";
            }
        }
    }
    
    private function tryHomepage() {
        if ($this->post_count > 0) return;
        
        echo "<p>🔗 Scanning homepage...</p>";
        
        $html = @file_get_contents($this->website_url);
        if (!$html) {
            echo "<p style='color: red;'>❌ Cannot access homepage</p>";
            return;
        }
        
        echo "<p style='color: green;'>✅ Homepage accessed</p>";
        
        // Look for date patterns in HTML
        $patterns = [
            '/\d{4}-\d{2}-\d{2}/', // YYYY-MM-DD
            '/\d{2}\/\d{2}\/\d{4}/', // DD/MM/YYYY
            '/\d{2}-\d{2}-\d{4}/' // DD-MM-YYYY
        ];
        
        $today_matches = 0;
        foreach ($patterns as $pattern) {
            preg_match_all($pattern, $html, $matches);
            foreach ($matches[0] as $match) {
                if ($this->isToday($match)) {
                    $today_matches++;
                }
            }
        }
        
        if ($today_matches > 0) {
            $this->post_count = $today_matches;
            echo "<p style='color: blue;'>📊 Found approximately {$today_matches} posts from today on homepage</p>";
        }
    }
    
    private function isToday($date_string) {
        if (!$date_string) return false;
        
        try {
            $date = new DateTime($date_string);
            $date_formatted = $date->format('Y-m-d');
            return $date_formatted === $this->today;
        } catch (Exception $e) {
            return false;
        }
    }
    
    public function displayResults() {
        echo "<div style='background: #f0f0f0; padding: 20px; border-radius: 10px; margin-top: 20px;'>";
        echo "<h3>📊 FINAL RESULTS</h3>";
        echo "<p><strong>🌐 Website:</strong> {$this->domain}</p>";
        echo "<p><strong>📅 Date:</strong> {$this->today}</p>";
        echo "<p><strong>🔗 URL:</strong> {$this->website_url}</p>";
        echo "<hr>";
        
        if ($this->post_count > 0) {
            echo "<h2 style='color: green;'>✅ Today's Posts: {$this->post_count}</h2>";
            echo "<p>This website published <strong>{$this->post_count}</strong> posts today.</p>";
        } else {
            echo "<h2 style='color: orange;'>⚠️ No posts found for today</h2>";
            echo "<p>Either no posts were published today, or we couldn't detect them.</p>";
        }
        
        echo "</div>";
    }
}

// HTML Interface
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Website Daily Post Counter</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        
        body {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: white;
            border-radius: 15px;
            padding: 30px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
        }
        
        h1 {
            color: #333;
            text-align: center;
            margin-bottom: 30px;
            background: linear-gradient(45deg, #667eea, #764ba2);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
        
        .form-group {
            margin-bottom: 20px;
        }
        
        label {
            display: block;
            margin-bottom: 8px;
            font-weight: 600;
            color: #555;
        }
        
        input[type="url"] {
            width: 100%;
            padding: 15px;
            border: 2px solid #ddd;
            border-radius: 8px;
            font-size: 16px;
            transition: border-color 0.3s;
        }
        
        input[type="url"]:focus {
            border-color: #667eea;
            outline: none;
        }
        
        button {
            background: linear-gradient(45deg, #667eea, #764ba2);
            color: white;
            border: none;
            padding: 15px 30px;
            border-radius: 8px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            width: 100%;
            transition: transform 0.3s;
        }
        
        button:hover {
            transform: translateY(-2px);
        }
        
        .results {
            margin-top: 30px;
            padding: 20px;
            background: #f8f9fa;
            border-radius: 10px;
            border-left: 5px solid #667eea;
        }
        
        .loading {
            display: none;
            text-align: center;
            margin: 20px 0;
        }
        
        .example {
            background: #e8f4f8;
            padding: 15px;
            border-radius: 8px;
            margin-top: 20px;
            font-size: 14px;
        }
        
        .example h3 {
            color: #2c3e50;
            margin-bottom: 10px;
        }
        
        .example ul {
            list-style: none;
            padding-left: 0;
        }
        
        .example li {
            padding: 5px 0;
            color: #555;
        }
        
        @media (max-width: 600px) {
            .container {
                padding: 20px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🌐 Website Daily Post Counter</h1>
        
        <form method="POST" action="">
            <div class="form-group">
                <label for="website_url">Enter Website URL:</label>
                <input type="url" 
                       id="website_url" 
                       name="website_url" 
                       placeholder="https://example.com or example.com" 
                       value="<?php echo isset($_POST['website_url']) ? htmlspecialchars($_POST['website_url']) : ''; ?>"
                       required>
            </div>
            
            <button type="submit" name="check_posts">
                🔍 Check Today's Posts
            </button>
        </form>
        
        <div class="loading" id="loading">
            <p>⏳ Scanning website, please wait...</p>
        </div>
        
        <?php
        if (isset($_POST['check_posts']) && !empty($_POST['website_url'])) {
            $url = trim($_POST['website_url']);
            
            echo '<div class="results">';
            
            try {
                $counter = new WebsitePostCounter($url);
                $counter->checkTodayPosts();
                $counter->displayResults();
            } catch (Exception $e) {
                echo "<p style='color: red;'>❌ Error: " . $e->getMessage() . "</p>";
            }
            
            echo '</div>';
        }
        ?>
        
        <div class="example">
            <h3>📋 Examples to Try:</h3>
            <ul>
                <li>🔗 <strong>https://bbc.com</strong> - News website</li>
                <li>🔗 <strong>https://techcrunch.com</strong> - Tech blog</li>
                <li>🔗 <strong>https://wordpress.org</strong> - Blog platform</li>
                <li>🔗 <strong>https://github.blog</strong> - Development blog</li>
            </ul>
            <p><small>Note: Some websites may block automated access or not provide dates.</small></p>
        </div>
    </div>
    
    <script>
        document.querySelector('form').addEventListener('submit', function() {
            document.getElementById('loading').style.display = 'block';
        });
    </script>
</body>
</html>
