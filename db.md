先从一个最简单的查询数据库数据说起，控制器代码如下
```
class TestController extends Controller
{
  public function actionDb()
  {
    $db = new \yii\db\Connection([
		    'dsn' => 'mysql:host=192.168.124.10;dbname=test',
		    'username' => 'root',
		    'password' => '',
		    'charset' => 'utf8',
		]);
		$data = $db->createCommand('SELECT * FROM a')
            ->queryAll();
    $this->asJson($data);
  }
}
```
