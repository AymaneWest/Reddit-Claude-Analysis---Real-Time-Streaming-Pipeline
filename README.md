# Reddit Claude Analysis - Real-Time Streaming Pipeline

##  Project Overview

This project implements a **real-time data engineering pipeline** that:
- Streams Reddit posts and comments mentioning Claude AI (and competitors)
- Processes data with sentiment analysis using Kafka
- Builds a dimensional data warehouse in Hive/Cloudera
- Enables competitive analysis and sentiment tracking

### Architecture Flow
```
Reddit API → Python Producer → Kafka Topics → Python Consumer → 
Sentiment Analysis → Star Schema → SFTP → Cloudera/Hive → Power BI
```

##  Tech Stack

- **Data Collection**: Python (PRAW - Reddit API)
- **Stream Processing**: Apache Kafka
- **Data Processing**: Python (Pandas, Transformers, NLTK)
- **Sentiment Analysis**: HuggingFace Transformers (DistilBERT)
- **File Transfer**: SFTP (Paramiko)
- **Data Warehouse**: Apache Hive on Cloudera
- **Visualization**: Power BI (or Tableau)

##  Prerequisites

### System Requirements
- Ubuntu 20.04+ or similar Linux distribution
- Python 3.8+
- Apache Kafka 2.8+
- Cloudera VM (or standalone Hive)
- 8GB+ RAM recommended
- 20GB+ free disk space

### Reddit API Access
1. Go to https://www.reddit.com/prefs/apps
2. Create a new application (script type)
3. Note down: `client_id`, `client_secret`
4. Save your Reddit username and password

### Kafka Installation
```bash
# Download Kafka
wget https://downloads.apache.org/kafka/3.6.0/kafka_2.13-3.6.0.tgz
tar -xzf kafka_2.13-3.6.0.tgz
sudo mv kafka_2.13-3.6.0 /opt/kafka

# Add to PATH (add to ~/.bashrc)
export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

##  Quick Start

### 1. Clone and Setup
```bash
cd /home/claude
git clone <your-repo> reddit-claude-analysis
cd reddit-claude-analysis

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure API Credentials
```bash
# Edit config/config.yaml
nano config/config.yaml

# Update these sections:
# - reddit: your Reddit API credentials
# - sftp: your Cloudera VM connection details
```

### 3. Start Kafka Services
```bash
# Terminal 1 - Start Zookeeper
cd /opt/kafka
bin/zookeeper-server-start.sh config/zookeeper.properties

# Terminal 2 - Start Kafka
cd /opt/kafka
bin/kafka-server-start.sh config/server.properties
```

### 4. Run the Pipeline

#### Option A: Interactive Menu
```bash
./run_pipeline.sh
# Select options from menu
```

#### Option B: Full Automated Run
```bash
./run_pipeline.sh --full
```

#### Option C: Manual Step-by-Step
```bash
# Terminal 1 - Start Producer
python3 producers/reddit_producer.py

# Terminal 2 - Start Consumer
python3 consumers/reddit_consumer.py

# Wait for data collection (recommended: 2-24 hours)

# Terminal 3 - Generate Star Schema
python3 processors/star_schema_generator.py

# Terminal 4 - Transfer to Cloudera
python3 export/sftp_transfer.py
```

##  Data Flow Details

### Stage 1: Reddit Data Collection
**Script**: `producers/reddit_producer.py`

- Monitors 9+ subreddits in real-time
- Filters posts/comments containing keywords
- Streams to Kafka topics:
  - `reddit-claude-posts-raw`
  - `reddit-claude-comments-raw`

**Key Features**:
- Real-time streaming (not batch)
- Automatic rate limiting
- Keyword extraction
- Error handling & retry logic

### Stage 2: Data Processing & Enrichment
**Script**: `consumers/reddit_consumer.py`

- Consumes from Kafka topics
- Performs sentiment analysis (DistilBERT)
- Extracts topics (coding, writing, comparison, etc.)
- Calculates engagement scores
- Saves processed batches as CSV

**Enrichment**:
- Sentiment: label, score, polarity, subjectivity
- Topics: multi-label classification
- AI Model categorization
- Text cleaning & normalization

### Stage 3: Dimensional Modeling
**Script**: `processors/star_schema_generator.py`

Generates star schema with:

**Fact Table**:
- `fact_discussions`: 15+ metrics per discussion

**Dimension Tables**:
- `dim_date`: temporal attributes
- `dim_subreddit`: community metadata
- `dim_ai_model`: competitive tracking
- `dim_sentiment`: emotion classification
- `dim_topic`: discussion themes
- `dim_author`: user information

### Stage 4: Data Warehouse Loading
**Script**: `export/sftp_transfer.py`

- Securely transfers CSVs to Cloudera VM
- Verifies file integrity
- Prepares for Hive ingestion

**Hive Schema**: `schemas/hive_schema.sql`
- Creates database and tables
- Loads data from CSVs
- Defines analytical views

##  Key Analytical Queries

```sql
-- 1. Claude vs Competitors Sentiment
SELECT 
    m.model_name,
    AVG(f.sentiment_score) as avg_sentiment,
    COUNT(*) as mention_count
FROM fact_discussions f
JOIN dim_ai_model m ON f.ai_model_id = m.ai_model_id
GROUP BY m.model_name
ORDER BY avg_sentiment DESC;

-- 2. Daily Sentiment Trend
SELECT 
    d.full_date,
    AVG(f.polarity) as avg_polarity,
    COUNT(*) as discussion_count
FROM fact_discussions f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_ai_model m ON f.ai_model_id = m.ai_model_id
WHERE m.model_name = 'Claude'
GROUP BY d.full_date
ORDER BY d.full_date;

-- 3. Topic Distribution
SELECT 
    t.topic_name,
    COUNT(*) as count,
    AVG(f.sentiment_score) as avg_sentiment
FROM fact_discussions f
JOIN dim_topic t ON f.topic_id = t.topic_id
GROUP BY t.topic_name
ORDER BY count DESC;
```

##  Power BI Dashboard

### Connecting to Hive
1. Open Power BI Desktop
2. Get Data → More → Hive LLAP
3. Server: `192.168.234.152:10000`
4. Database: `reddit_claude_analytics`
5. Import or DirectQuery mode

### Recommended Visualizations

**Page 1: Executive Summary**
- KPI Cards: Total discussions, average sentiment, trending topics
- Line Chart: Sentiment trend over time
- Bar Chart: Claude vs Competitors

**Page 2: Competitive Analysis**
- Scatter Plot: Sentiment vs Engagement by AI model
- Stacked Bar: Topic distribution per model
- Table: Top discussions by engagement

**Page 3: Community Insights**
- Map/Heatmap: Subreddit sentiment
- Word Cloud: Common topics
- Timeline: Discussion volume by hour/day

**Page 4: Deep Dive**
- Drill-through pages for specific subreddits
- Filter by date range, sentiment, topics
- Export capabilities

##  Configuration Options

### Monitoring More Subreddits
Edit `config/config.yaml`:
```yaml
subreddits:
  - "ClaudeAI"
  - "artificial"
  - "YourCustomSubreddit"  # Add here
```

### Changing Keywords
```yaml
keywords:
  primary:
    - "Claude"
    - "Your Custom Keyword"
```

### Adjusting Processing
```yaml
processing:
  batch_size: 100  # Increase for better performance
  processing_interval_seconds: 60
```

##  Monitoring & Logs

### Check Pipeline Status
```bash
# View logs
tail -f logs/pipeline_*.log
tail -f logs/producer_*.log
tail -f logs/consumer_*.log

# Check Kafka topics
kafka-topics.sh --bootstrap-server localhost:9092 --list

# View topic contents (debugging)
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic reddit-claude-posts-raw --from-beginning
```

### Performance Metrics
- Producer rate: ~10-50 messages/minute (depends on Reddit activity)
- Consumer processing: ~100-200 messages/minute
- Sentiment analysis: ~5-10 messages/second

##  Troubleshooting

### Issue: Reddit API Rate Limits
**Solution**: Increase sleep intervals in `reddit_producer.py`

### Issue: Kafka Connection Refused
**Solution**: 
```bash
# Check if Kafka is running
ps aux | grep kafka

# Restart services
./run_pipeline.sh
# Select option 6 (Stop services), then restart
```

### Issue: Sentiment Analysis Out of Memory
**Solution**: Reduce batch size in config or use GPU if available

### Issue: SFTP Transfer Fails
**Solution**: 
- Verify Cloudera VM is accessible: `ping 192.168.234.152`
- Check SSH credentials
- Ensure remote directory exists

##  Project Structure
```
reddit-claude-analysis/
├── config/
│   └── config.yaml              # Configuration file
├── producers/
│   └── reddit_producer.py       # Kafka producer
├── consumers/
│   └── reddit_consumer.py       # Kafka consumer
├── processors/
│   └── star_schema_generator.py # Dimensional modeling
├── export/
│   └── sftp_transfer.py         # Data transfer
├── schemas/
│   └── hive_schema.sql          # Hive DDL
├── data/
│   ├── raw/                     # Raw Kafka data
│   ├── processed/               # Processed CSVs
│   └── export/                  # Star schema CSVs
├── logs/                        # Application logs
├── requirements.txt             # Python dependencies
├── run_pipeline.sh              # Orchestration script
└── README.md                    # This file
```

##  Learning Outcomes

This project demonstrates:
1. **Real-time streaming** with Kafka
2. **ETL pipeline** design
3. **NLP & sentiment analysis**
4. **Dimensional modeling** (star schema)
5. **Data warehouse** implementation
6. **Distributed systems** coordination
7. **API integration** (Reddit, SFTP)
8. **Production-ready** error handling

##  Future Enhancements

- [ ] Add Apache Airflow for scheduling
- [ ] Implement Spark for large-scale processing
- [ ] Add PostgreSQL for operational data store
- [ ] Create Docker containers for portability
- [ ] Add monitoring with Prometheus + Grafana
- [ ] Implement data quality checks (Great Expectations)
- [ ] Add incremental loading logic
- [ ] Create REST API for data access

##  License

MIT License - Feel free to use for learning and portfolio projects

##  Author

Built as a data engineering portfolio project demonstrating end-to-end pipeline development.

##  Acknowledgments

- Reddit API (PRAW library)
- Apache Kafka community
- HuggingFace Transformers
- Cloudera/Hive ecosystem
