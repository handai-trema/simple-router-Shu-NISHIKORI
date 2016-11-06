###どうでもいいですが，
予習したいので次回の内容をアップロードしていただくことはできませんでしょうか．

##解答
書き加えたソースコードは以下である．  
* [simple_router.rb](https://github.com/handai-trema/simple-router-Shu-NISHIKORI/blob/develop/report/simple_router.rb)  
* [routing_table.rb](https://github.com/handai-trema/simple-router-Shu-NISHIKORI/blob/develop/report/routing_table.rb)  
* [routing_table](https://github.com/handai-trema/simple-router-Shu-NISHIKORI/blob/develop/report/routing_table)  

###ルーティングテーブルの表示
書き加えたコードは以下である．
```
routing_table
  include Pio
  desc 'show routing table'
  command :show_db do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      puts("routing table")
      db = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        show_db
      db.each do |each|
        each.each_key do |key|
          print "Dest : ", IPv4Address.new(key).to_s, "/", key.to_i, "\n"
          print "Next : ", each[key].to_s, "\n"
        end
      end
    end
  end


simple_router.rb
  def show_db
    return @routing_table.db
  end


routing_table.rb
  attr_accessor :db
```
経路表が格納されているのはrouting_table.rb内のdbである．しかし，デフォルトでbin内に作ったrouting_tableから参照できるのはsimple_router.rbのみのようなので，いったんsimple_router.rbを経由してdbを取得した．取得の仕方としては，attr_accessor :dbによってdbの外部からのアクセス権限を与えることによって実現した．  
表示について，dbの各ネットマスクの各ハッシュのkeyにdestination addressが，valueにnext hip addressが入っている．しかし，格納時に整数型に変換されているようなので，表示時にto_sで文字列に戻した．なお，IPv4Addressのモジュールを使用するのにPioをincludeしなければならないようなので書き加えている．


###ルーティングテーブルエントリの追加と削除
書き加えたコードは以下である．
```
routing_table
  desc 'add forwardind entry'
  arg_name 'dest netmask next'
  command :add do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      add_db_entry(args[0], args[1], args[2])
    end
  end

  desc 'delete forwardind entry'
  arg_name 'dest netmask'
  command :delete do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      delete_db_entry(args[0], args[1], nil)
    end
  end


simple_router.rb
  def add_db_entry(destination_ip, netmask_length, next_hop)
    options = {:destination => destination_ip, :netmask_length => netmask_length.to_i, :next_hop => next_hop}
    @routing_table.add(options)
  end

  def delete_db_entry(destination_ip, netmask_length, next_hop)
    options = {:destination => destination_ip, :netmask_length => netmask_length.to_i, :next_hop => next_hop}
    @routing_table.delete(options)
  end


routing_table.rb
  def add(options)	#改変なし
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length][prefix.to_i] = IPv4Address.new(options.fetch(:next_hop))
  end

  def delete(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length].delete(prefix.to_i)
  end
```
追加については，最初からrouting_table.rbにあるaddメソッドを利用した．addメソッド内で使用されている変数はnetmask_length, destination, next_hopの3つなので，これを実行コマンドの引数とした．ただし，netmask_lengthに関しては，db配列のインデックスに使用されているようなので，routing_table.rbに渡す際にto_iで整数型に変換している．  
削除についても追加と同様に作成した．ただし，転送エントリを削除する際に次ホップを指定する必要はないと考えたので，引数としてnext_hopはなく，代わりにsimple_router.rbに渡す際のnext_hopの中身をnilとした．

###ルータのインターフェース一覧の表示
[成元くんのレポート](https://github.com/handai-trema/simple-router-r-narimoto/blob/master/report.md)を参考にしました．ありがとうございました．
```
routing_table
  desc 'show interfaces'
  command :show_interfaces do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      interfaces = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      show_interface
      interfaces.each do |each|
        print "port : ", each[:port_number].to_s, ", MAC : ", each[:mac_address], ", IP : ", each[:ip_address], "/", each[:netmask_length].to_s, "\n"
      end
    end
  end


simple_router.rb
  def show_interface
    ret = Array.new()
    Interface.all.each do |each|
      ret.push(:port_number => each.port_number, :mac_address => each.mac_address.to_s, :ip_address => each.ip_address.value.to_s, :netmask_length => each.netmask_length)
    end
    return ret
  end
```
インターフェースの中身には，Interfaceの変数port_number， mac_address， ip_address， netmask_lengthからアクセスできる．Interfaceの全要素について保持する変数retを用意し，各ポートごとの情報を格納させた．そしてそれをrouting_tableに返して，routing_tableで走査させることで表示させている．


###動作確認
simple_router.rbの実行の際には，付属のtrema.confを設定ファイルとして指定している．
```
simple_router.rb
vswitch('0x1') { dpid 0x1 }
netns('host1') {
  ip '192.168.1.2'
  netmask '255.255.255.0'
  route net: '0.0.0.0', gateway: '192.168.1.1'
}
netns('host2') {
  ip '192.168.2.2'
  netmask '255.255.255.0'
  route net: '0.0.0.0', gateway: '192.168.2.1'
}
link '0x1', 'host1'
link '0x1', 'host2'
```
```
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table show
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table add 192.168.3.2 24 192.168.2.2
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table show
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2
Dest : 192.168.3.2/24
Next : 192.168.2.2
$ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table delete 192.168.3.2 24
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table show
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table show_interface
port : 1, MAC : 01:01:01:01:01:01, IP : 192.168.1.1/24
port : 2, MAC : 02:02:02:02:02:02, IP : 192.168.2.1/24
```
.confファイルの内容通りに表示されており，追加/削除したエントリも反映されていることを確認した．