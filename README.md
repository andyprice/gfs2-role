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

- community.general.lvol
- community.general.lvg

Example Playbook
----------------

    - hosts: cluster-virts
      vars:
        gfs2_cluster_name: MyCluster
      pre_tasks:
        - name: Check whether cluster has been set up yet
          # A more robust check may be required
          ansible.builtin.command: pcs cluster status
          register: cluster_exists
          changed_when: false
          failed_when: false

        - name: Create cluster if it doesn't exist
          ansible.builtin.include_role:
            name: linux-system-roles.ha_cluster
          vars:
            ha_cluster_cluster_name: "{{ gfs2_cluster_name }}"
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
          when: cluster_exists.rc != 0
    
      roles:
        - role: gfs2
          vars:
            # Specify 2 gfs2 file systems
            gfs2_file_systems:
              - name: fs1
                pvs:
                  - /dev/disk/by-path/virtio-pci-0000:00:08.0
                vg: vg_gfs2_1
                lv: lv_gfs2_1
                lv_size: 100%VG
                mount_point: /mnt/test1
                mount_options:
                  - noatime
              - name: fs2
                pvs:
                  - /dev/disk/by-path/virtio-pci-0000:01:00.0
                vg: vg_gfs2_2
                lv: lv_gfs2_2
                lv_size: 100%VG
                mount_point: /mnt/test2
                mount_options:
                  - noatime

License
-------

MIT

Author Information
------------------

Andrew Price
