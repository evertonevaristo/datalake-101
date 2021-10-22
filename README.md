# Windows

Instalação do agente:

https://docs.fluentd.org/installation/install-by-msi#step-1-install-td-agent

Plugins necessários:

```
td-agent-gem install fluent-plugin-ec2-metadata 
td-agent-gem install fluent-plugin-windows-eventlog
```

Os plugins são necessários para ajudar em duas necessidades:

1. Obter os dados de metadado da instancia de origem;
2. Realizar o Parse dos logs XML do Windows

Exemplo de arquivo de configuração (td-agent.conf)

```
<system>
    @log_level trace
</system>

<source>
    @type windows_eventlog2
    @id winevtapplication
    channels application
    read_existing_events false
    tag winevtapplication.raw
    render_as_xml true
    rate_limit 200
    <storage>
        @type local
        persistent true
        path C:/opt/td-agent/winapplicationlog.json
    </storage>
    <parse>
        @type none
    </parse>
</source>

<filter winevtapplication>
    @type record_transformer
    enable_ruby true
    <record>
      current_date ${require 'date'; current_time = DateTime.now; current_time.strftime "year=%Y/month=%m/day=%d"}
    </record>
</filter>

<match winevtapplication.raw>
    @type rewrite_tag_filter
    <rule>
        key current_date
        pattern ^(.+)$
        tag dated.${tag}.$1
    </rule>
</match>
 
<match dated.winevtapplication.raw.**>
    @type ec2_metadata
    metadata_refresh_seconds 300 # Optional, default 300 seconds
    imdsv2 true                  # Optional, default false
    output_tag ${instance_id}.${account_id}.${tag}
    <record>
        instance_id   ${instance_id}
        account_id    ${account_id}
    </record>
</match>
 
<match **.**.dated.winevtapplication.raw.**>
    @type copy
    <store>
    @type s3
        use_server_side_encryption "aws:kms"
        time_slice_format %Y%m%d
        path ec2/windows/itau-aws-sor-saeast1
        s3_bucket sor-data-oyndzp
        s3_region sa-east-1
        s3_object_key_format "%{path}-${tag[1]}/${tag[0]}/${tag[3]}/${tag[5]}/${tag[3]}_%{time_slice}_%{index}.%{file_extension}" 
        <format>
            @type json
        </format>
        <buffer tag,time>
            @type file
            path C:/var/log/td-agent/s3
            timekey 360 # 1 hour partition
            timekey_wait 10m
            timekey_use_utc true # use utc
            chunk_limit_size 256m 
        </buffer>
</store>

<store>
    @type stdout
</store>    
 
</match>
```

#### Etapas do processamento no Fluentd:

1. O plugin windows_eventlog2 capta os logs de acordo com o especificado, e insere uma tag

_log = application, tag = winapplication_

2. Para cada evento capturado, um filtro é aplicado para inserir a data atual;

_year=2021/month=10/day=21_

3. A data capturada no passo anterior, e o prefixo "dated" são inseridas no evento atual.

_dated.winapplication.year=2021/month=10/day=21_

4. O plugin ec2_metadata adiciona os campos instance-id e account-id na tag do evento atual

_instance-id.account-id.dated.winapplication.year=2021/month=10/day=21_

5. O log é enviado para o s3, usando o plugin de output. 

_instance-id.account-id.dated.winapplication.year=2021/month=10/day=21.message{...}_

# Linux

Instalação do agente:

https://docs.fluentd.org/installation/install-by-rpm#amazon-linux

Os plugins são necessários para ajudar na seguinte necessidade:

1. Obter os dados de metadado da instancia de origem;

_$ sudo /usr/sbin/td-agent-gem install fluent-plugin-ec2-metadata_

Exemplo de arquivo de configuração (td-agent.conf)

```
<source>
  @type tail
  path /etc/audit/auditd.conf,/etc/audit/rules.d/itau-cybersecurity.rules,/etc/audisp/plugins.d/syslog.conf,/etc/rsyslog.conf,/etc/awslogs/awslogs.conf,/var/log/security,/var/log/audit/audit.log
  pos_file /var/log/td-agent/auditlog.pos
  read_from_head true
  follow_inodes true
  tag auditlog.*
  <parse>
   @type none
  </parse>
</source>

<match auditlog.**>
  @type ec2_metadata
  metadata_refresh_seconds 300 # Optional, default 300 seconds
  imdsv2 true                  # Optional, default false
  output_tag ${instance_id}.${account_id}.${tag}
  <record>
    instance_id   ${instance_id}
    az            ${availability_zone}
    account_id    ${account_id}
  </record>
</match>

<match **.**.auditlog.**>
 
  @type copy
    @type s3
        use_server_side_encryption "aws:kms"
        time_slice_format %Y%m%d
        path ec2/linux/itau-aws-sor-saeast1
        s3_bucket sor-data-oyndzp
        s3_region sa-east-1
        s3_object_key_format "%{path}-${tag[1]}/${tag[0]}/${tag[2]}/#{%x`printf '%(year=%Y/month=%m/day=%d)T' -1`}/${tag[2]}_%{time_slice}_%{index}.%{file_extension}"
    <format>
        @type json
    </format>
 
    <buffer tag,time>
    @type file
        path /var/log/td-agent/buffer/s3
        timekey 3600 # 1 hour partition
        timekey_wait 10m
        timekey_use_utc false # use utc
        chunk_limit_size 256m
    </buffer>
 
</match>
```

#### Etapas do processamento no Fluentd:


1. A origem dos arquivos é configurada e uma tag de auditlog é inserida nos eventos:

_log = /var/log/teste_
_auditlog.var.log.teste_

2. O plugin ec2_metadata adiciona os campos instance-id e account-id na tag do evento atual

_instance-id.account-id.auditlog.var.log.teste_

3. O log é enviado para o s3, usando o plugin de output. 

_instance-id.account-id.auditlog.var.log.teste.year=2021/month=10/day=21.message{...}_

  
