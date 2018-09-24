# 概要
SageMakerからS3のファイルを操作するためのcloudforamtionテンプレートを作成した。
https://dev.classmethod.jp/machine-learning/amazon-sagemaker-glue-s3-emr-operation/

基本的には下記のエントリー通りに実行していけば問題ないが、これだと「SageMakerインスタンス」も再作成することになってしまう。  
https://aws.amazon.com/jp/blogs/news/access-amazon-s3-data-managed-by-aws-glue-data-catalog-from-amazon-sagemaker-notebooks/

そのため、「既存のSageMakerインスタンスに適用するため」のテンプレートを用意した。


### emr_glue_s3.yamlの内容について
「EMRクラスター」「Glueクローラー」を作成する。
「既存のSageMakerインスタンス」のライフサイクル設定に下記のような処理を入れてあげると、ここで作成した「EMRクラスター」と接続できるようになる。
もし接続できないようなら、「既存のSageMakerインスタンス」のIAMロールに、EMRにアクセスできる権限（ポリシー）が付与されているかを確認すること。


```
#!/bin/bash
#set -e         
EMRGLUEIP=`aws emr list-instances --cluster-id <EMRクラスターのID> --instance-group-types MASTER --region <EMRクラスターがあるリージョン> | jq -r '.Instances[0].PrivateIpAddress' `
 
echo $(date +%F_%T) 'Using: <EMRクラスターのID>: ' $EMRGLUEIP
 
wget -O /home/ec2-user/SageMaker/.sparkmagic/config.json https://raw.githubusercontent.com/jupyter-incubator/sparkmagic/master/sparkmagic/example_config.json 
sed -i -e "s/localhost/$EMRGLUEIP/g" /home/ec2-user/SageMaker/.sparkmagic/config.json
# Next line needed to bypass 'failed to register auto_viz' error in recent version
sed -i -e 's/"use_auto_viz": true/"use_auto_viz": false/g' /home/ec2-user/SageMaker/.sparkmagic/config.json

```


### SageMakerでの操作
下記の3段階。

##### 1.sparkとのセッションを確保する

```
import os
import platform
# Print some characteristics of the remote system
print(platform.node())
print(platform.platform(aliased=0, terse=0))
print("Spark home currently set to", os.environ.get('SPARK_HOME', None))
```

##### 2.Glueデータカタログに指定したデータに対してクエリを実行する
「-o」引数に結果を格納する。


```
%%sql -o members
select p.name as membername, p.gender, p.birth_date, p.death_date, 
    m.on_behalf_of_id, m.legislative_period_id, m.start_date, m.end_date, 
    o.name as orgname, o.classification
from legislators.persons_json p
join legislators.memberships_json m on m.person_id = p.id
join legislators.organizations_json o on m.organization_id = o.id

```


##### 3.取得した結果をローカルで利用する
「-o」引数に格納した結果を、SageMakerインスタンスのローカルで利用可能。

```
%%local
membersadj['gen'] = membersadj['gender'].map({'female': 1, 'male': 0})
membersadj['partynum'] = membersadj['party'].map({
    'republican-conservative':0, 
    'al':1,
    'republican':2,
    'democrat': 3,
    'popular_democrat':4,
    'new_progressive':5,
    'democrat-liberal':6,
    'independent':7  })
 
membersadj['legstart'] =(membersadj['legislature']*2 + 1767)
membersadj['birthyear'] = membersadj['birth_date'].dt.year
membersadj['age'] = membersadj['legstart'] - membersadj['birthyear']
 
membersadj.head()

```



