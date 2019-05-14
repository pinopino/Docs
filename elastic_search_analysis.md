Elasticsearch�е�Mapping��Analysis
=========

## Analysis

����analysis����Ҫ��ʶ��һ�㣬����һ���ĵ��Լ���ѯ��ʱ����ʵ������Ҫ�õ��ִ����ģ����ʱ��ͨ���������õķִ���Ӧ��Ҫһ�²źá�

�����ĵ���ʱ���������û��ָ���κηִ�������ôĬ��ES�ͻ�ʹ��`standard analyzer`������ѯʱ�ִ�����ȷ��ES��һ�����ȼ���
- An analyzer specified in the query itself
- The search_analyzer mapping parameter
- The analyzer mapping parameter
- An analyzer in the index settings called default_search
- An analyzer in the index settings called default
- The standard analyzer

�ܹ��������������ʲô��û��ָ���Ļ�Ĭ��ֵҲ����`standard analyzer`�������⣬��ÿһ�εĲ�ѯ��ֱ��ָ��analyzer���Ἣ��ķ������ǲ��Բ�ѯ��
```csharp
// ����Ҫȥָ����������Ϊ��ֻ��һ�������Ĳ�����䣬�����������޹�
// ����ÿ�ζ����ò�ͬ��analyzer����Ч��
POST /_analyze
{
  "analyzer": "whitespace",
  "text":     "The quick brown fox."
}
```

�����б�Ҫ��������`analyzer`����ES�����Ķ���ܼ򵥣������������ִ����ɵ�һ�����壺
- �ַ���������character filters��
���磬�ܹ�ȥ��HTML����ֻ���ת�� "&" Ϊ "and"�����⣬����ַ����������Դ���������˳��ִ�У�
- �ִ�����tokenizers��
�����ִ����ķִ����ǲ��ܵõ�һ������token���Թ���һ����ǹ�����ʹ�ã������ַ���������һ��analyzerֻ�ܺ���һ���ִ���
- ��ǹ�������token filters��
���磬`lowercase` token filter�ܹ�������tokenȫ��ת��ΪСд���ֻ��ߣ�һ��`stop` token filter�ܹ�ɾ��token���г�����ͣ�ʣ���ǹ�����Ҳ�ܹ��������������˳��ִ�У�

����ζ�Ź���`analyzer`���ǿ��������滻�����е�����������������������Լ���`custom analyzer`��
```csharp
// ע����Ϊ���Զ���ģ���������type����ֵһ����"custom"
// ע��analyzer�������Ƿ���������"settings"���еģ���û����"mapping"��
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type":      "custom", 
          "tokenizer": "standard",
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "Is this <b>d��j�� vu</b>?"
}

// �������������Զ�����������ֱ���и���ϸ������
// ��ʵ����Ҳ�������ǣ���ʹ��һЩ������analyzerʱ������ȥ������Щʲô
// ���ÿ��Զ��Ƶ�
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": [
            "emoticons" 
          ],
          "tokenizer": "punctuation", 
          "filter": [
            "lowercase",
            "english_stop" 
          ]
        }
      },
      "tokenizer": {
        "punctuation": { 
          "type": "pattern",
          "pattern": "[ .,!?]"
        }
      },
      "char_filter": {
        "emoticons": { 
          "type": "mapping",
          "mappings": [
            ":) => _happy_",
            ":( => _sad_"
          ]
        }
      },
      "filter": {
        "english_stop": { 
          "type": "stop",
          "stopwords": "_english_"
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text":     "I'm a :) person, and you?"
}
```
��������ES�Դ���analyzer��������Ҳ�ܹ��ã�������Щ�е���ʦ�����ˣ���ô����������������
```csharp
// ����������һ����Ϊ"std_english"��analyzer�������������Ͳ�����custom��
// ���������"standard"������ζ�����ǽ����ǻ���standard����һЩ�Զ������
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",
          "stopwords": "_english_" // "stopwords"��standard analyzer
          // ���е�һ���������ԣ����������������ʹ��������һЩ���ƣ�
          // ��Ҳ�������ǣ����ʹ��������������analyzer����ȥ��ע��������
          // ������Щ�ɶ�������
        }
      }
    }
  },
  "mappings": {
    "_doc": {
      "properties": {
        "my_text": {
          "type":     "text",
          "analyzer": "standard", 
          "fields": {
            "english": {
              "type":     "text",
              "analyzer": "std_english" 
            }
          }
        }
      }
    }
  }
}

POST my_index/_analyze
{
  "field": "my_text", 
  "text": "The old brown cow"
}

POST my_index/_analyze
{
  "field": "my_text.english", 
  "text": "The old brown cow"
}
```
������δ���Ҳ��ʾ�����Ϊ`multi-fields` ָ����ͬ��analyzer��ֵ��һ����


