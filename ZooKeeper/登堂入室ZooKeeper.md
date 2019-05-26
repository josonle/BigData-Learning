### Java API使用

```java
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;

import java.io.IOException;
import java.util.List;

public class CreateConnector {
	private static String addr = "master:2181,slave1:2181,slave2:2181";
	private ZooKeeper zKeeper = null;

	public void getConnect() throws IOException, KeeperException, InterruptedException {
		zKeeper = new ZooKeeper(addr, 2000, null);
//		new ZooKeeper(addr, 2000, new Watcher() {//传入一个监听器
//
//			public void process(WatchedEvent event) {//znode发生变更时如何处理逻辑
//				// TODO Auto-generated method stub
//				
//			}
//		});
	}

	public void createZNode(String path,String value) throws KeeperException, InterruptedException {
		if (zKeeper.exists(path, true) != null) {//判断znode是否存在，返回一个Stat
			Stat stat1 = zKeeper.setData(path, value.getBytes(), -1);//更新znode
			System.out.println(stat1.getCversion()+","+stat1.getVersion()+","+stat1.getMtime());//打印stat信息
		} else {//创建znode
			String path1 = zKeeper.create(path, value.getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
			System.out.println(path);
		}
	}

	private static Stat stat = new Stat();

	public void pringChildren(String str) throws KeeperException, InterruptedException {
		List<String> childList = zKeeper.getChildren(str, true);
		if(childList.isEmpty())
			return;
		for (String s : childList) {
			String childStr = str.equals("/")?"/"+s:str+"/"+s;
			System.out.print(childStr+ ":");
			System.out.println(new String(zKeeper.getData(childStr, false, stat)));
			pringChildren(childStr);
		}
	}

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		CreateConnector test = new CreateConnector();
		try {
			test.getConnect();
			test.pringChildren("/");
			test.createZNode("/ZKTest","testing");
			test.pringChildren("/");
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```

