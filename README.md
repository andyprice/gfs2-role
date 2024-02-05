gfs2
====

**Note: For now, this ansible role is a proof-of-concept and should be considered unstable.**

Configure and manage gfs2 file systems in a pacemaker cluster.

Requirements
------------


Role Variables
--------------

See [meta/argument_specs.yml](meta/argument_specs.yml)

Dependencies
------------

- linux-system-roles.storage

Example Playbooks
-----------------

### Minimal example

```yaml
- hosts: myapp-servers
  roles:
    - role: gfs2
      vars:
        gfs2_cluster_name: myapp-cluster
        gfs2_file_systems:
          - name: myapp-shared-fs
            pvs:
              - /dev/disk/by-path/pci-0000:42:00.0-fc-0xf000-lun-1
            vg: vg_myapp_shared
            lv: lv_myapp_shared
            lv_size: 100G
            mount_point: /mnt/myapp-shared
```

### Minimal example with cluster setup

```yaml
- hosts: cluster-virts
  vars:
    common_cluster_name: MyCluster
  pre_tasks:
    - name: Check whether cluster has been set up yet
      # We do this because the ha_cluster role removes any resources not specified
      # by its variables. It is safer to run ha_cluster in a separate one-off
      # playbook which will not be used again after the gfs2 resources are created.
      ansible.builtin.command: pcs stonith status
      register: cluster_exists
      changed_when: false
      failed_when: false

    - name: Create cluster if it doesn't exist
      ansible.builtin.include_role:
        name: linux-system-roles.ha_cluster
      vars:
        ha_cluster_cluster_name: "{{ common_cluster_name }}"
        ha_cluster_enable_repos: no
        # Users should vault-encrypt the password
        ha_cluster_hacluster_password: hunter2
        ha_cluster_fence_virt_key_src: fence_xvm.key
        ha_cluster_cluster_properties:
          - attrs:
            - name: stonith-enabled
              value: 'true'
        ha_cluster_resource_primitives:
          - id: xvm-fencing
            agent: 'stonith:fence_xvm'
      when:
        - cluster_exists.rc != 0
          or "Started" not in cluster_exists.stdout

  roles:
    - role: gfs2
      vars:
        gfs2_cluster_name: "{{ common_cluster_name }}"
        # Specify 2 gfs2 file systems
        gfs2_file_systems:
          - name: fs1
            pvs:
              - /dev/disk/by-path/virtio-pci-0000:00:08.0
            vg: vg_gfs2_1
            lv: lv_gfs2_1
            lv_size: 100G
            mount_point: /mnt/test1
          - name: fs2
            pvs:
              - /dev/disk/by-path/virtio-pci-0000:01:00.0
            vg: vg_gfs2_2
            lv: lv_gfs2_2
            lv_size: 100G
            mount_point: /mnt/test2
```

### Example including optional role variables

```yaml
- hosts: cluster-virts

  # (Cluster setup omitted)

  roles:
    - role: gfs2
      vars:
        gfs2_cluster_name: MyCluster
        gfs2_resource_name_lvmlockd: lvm_locking
        gfs2_resource_name_dlm: dlm_control
        gfs2_group_name_locking: locking
        gfs2_file_systems:
          - name: fs1
            resource_name_fs: gfs2-1
            pvs:
              - /dev/disk/by-path/virtio-pci-0000:00:08.0
            vg: vg_gfs2_1
            lv: lv_gfs2_1
            lv_size: 100G
            resource_name_lv: shared-lv-1
            journals: 3
            mount_point: /mnt/test1
            mount_options:
              - noatime
            state: enabled
          - name: fs2
            resource_name_fs: gfs2-2
            pvs:
              - /dev/disk/by-path/virtio-pci-0000:01:00.0
            vg: vg_gfs2_2
            lv: lv_gfs2_2
            lv_size: 100G
            resource_name_lv: shared-lv-2
            journals: 3
            mount_point: /mnt/test2
            mount_options:
              - rgrplvb
            state: disabled
```

License
-------

MIT

Author Information
------------------

Andrew Price
