# vm_service_types.proto — Examples

This document shows example configurations using the new `DevicesConfig` proto
schema. All examples use protobuf text format.

## Design Principles

1. **Bus owns addressing, device is what sits there** — the same `VirtioDevice`
   definition works on PCI or MMIO. The same `DiskConfig` works in VirtIO,
   NVMe, or SCSI.
2. **Maps enforce uniqueness** — PCI buses, root ports, VMBus devices, MMIO
   slots, and NVMe namespaces use map keys. Duplicates are impossible at the
   protocol level.
3. **Type-safe hot-plug** — each slot type only accepts device types valid for
   that bus. You cannot accidentally put a SCSI disk in a PCI slot.

---

## Initial VM Configuration Examples

### 1. VirtIO block device on PCI

A single PCI bus with one root port holding a VirtIO block device backed by a
VHDX image.

```protobuf
devices_config {
  pci_buses {
    key: "pci0"
    value {
      segment: 0
      root_ports {
        key: "port0"
        value {
          hotplug: true
          device {
            virtio {
              blk {
                disk {
                  host_path: "/var/vm/os-disk.vhdx"
                  format: DISK_FORMAT_VHDX
                }
                read_only: false
              }
            }
          }
        }
      }
    }
  }
}
```

### 2. VirtIO block device on MMIO (aarch64)

The same VirtIO block device, but attached via MMIO instead of PCI. Note how
the `VirtioDevice` message is identical — only the bus wrapper changes.

```protobuf
devices_config {
  mmio_buses {
    key: "mmio0"
    value {
      devices {
        key: 268435456  # 0x10000000
        value {
          irq: 42
          virtio {
            blk {
              disk {
                host_path: "/var/vm/os-disk.vhdx"
                format: DISK_FORMAT_VHDX
              }
              read_only: false
            }
          }
        }
      }
    }
  }
}
```

### 3. NVMe controller on PCI with two namespaces

```protobuf
devices_config {
  pci_buses {
    key: "pci0"
    value {
      segment: 0
      root_ports {
        key: "nvme-port"
        value {
          device {
            nvme {
              msix_count: 4
              max_io_queues: 4
              namespaces {
                key: 1
                value {
                  read_only: false
                  disk {
                    host_path: "/var/vm/data1.vhdx"
                    format: DISK_FORMAT_VHDX
                  }
                }
              }
              namespaces {
                key: 2
                value {
                  read_only: true
                  disk {
                    host_path: "/var/vm/data2.raw"
                    format: DISK_FORMAT_RAW
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 4. SCSI controller on VMBus with two disks

```protobuf
devices_config {
  vmbus_devices {
    key: "550e8400-e29b-41d4-a716-446655440000"
    value {
      scsi_controller {
        io_queue_depth: 128
        max_sub_channel_count: 4
        disks {
          key: 0
          value {
            read_only: false
            disk {
              host_path: "/var/vm/os-disk.vhdx"
              format: DISK_FORMAT_VHDX
            }
          }
        }
        disks {
          key: 1
          value {
            read_only: true
            disk {
              host_path: "/var/vm/data.vhd"
              format: DISK_FORMAT_VHD1
            }
          }
        }
      }
    }
  }
}
```

### 5. Serial console — three ways

The same serial backend (`socket`) is used across all three bus types.

**a) ISA COM port (chipset, x86)**

```protobuf
devices_config {
  chipset_devices {
    isa_serial {
      com_port: COM_PORT_COM1
      backend {
        socket { path: "/tmp/vm-serial.sock" }
      }
    }
  }
}
```

**b) UART on MMIO (aarch64)**

```protobuf
devices_config {
  mmio_buses {
    key: "mmio0"
    value {
      devices {
        key: 150994944  # 0x9000000 (typical PL011 base on aarch64)
        value {
          irq: 33
          uart {
            backend {
              socket { path: "/tmp/vm-serial.sock" }
            }
          }
        }
      }
    }
  }
}
```

**c) VirtIO console on PCI**

```protobuf
devices_config {
  pci_buses {
    key: "pci0"
    value {
      segment: 0
      root_ports {
        key: "console-port"
        value {
          device {
            virtio {
              console {
                backend {
                  socket { path: "/tmp/vm-serial.sock" }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 6. VirtIO network with consomme (NAT) backend on PCI

```protobuf
devices_config {
  pci_buses {
    key: "pci0"
    value {
      segment: 0
      root_ports {
        key: "net-port"
        value {
          device {
            virtio {
              net {
                mac_address: "12:34:56:78:9a:bc"
                max_queues: 4
                backend {
                  consomme {
                    cidr: "192.168.0.0/24"
                    port_forwards {
                      protocol: PORT_PROTOCOL_TCP
                      host_address: "127.0.0.1"
                      host_port: 2222
                      guest_port: 22
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 7. Synthetic NIC on VMBus (Hyper-V)

```protobuf
devices_config {
  vmbus_devices {
    key: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    value {
      synth_net {
        mac_address: "00:15:5d:01:02:03"
        max_queues: 8
        backend {
          dio {
            switch_id: "c08cb7b8-9b3c-408e-8e30-5e16a3aeb444"
            port_id: "d9f0e1a2-b3c4-5678-9012-3456789abcde"
          }
        }
      }
    }
  }
}
```

### 8. Full multi-device VM (x86 with PCI + VMBus + chipset)

```protobuf
devices_config {
  # PCI devices
  pci_buses {
    key: "pci0"
    value {
      segment: 0
      numa_node: 0
      root_ports {
        key: "blk0"
        value {
          device {
            virtio {
              blk {
                disk { host_path: "/data/os.vhdx" format: DISK_FORMAT_VHDX }
              }
            }
          }
        }
      }
      root_ports {
        key: "net0"
        value {
          hotplug: true
          device {
            virtio {
              net {
                mac_address: "aa:bb:cc:dd:ee:ff"
                max_queues: 2
                backend { consomme { cidr: "10.0.0.0/24" } }
              }
            }
          }
        }
      }
      root_ports {
        key: "hotplug0"
        value {
          hotplug: true
          # empty — ready for hot-plug
        }
      }
    }
  }

  # VMBus SCSI controller for data disks
  vmbus_devices {
    key: "ba6163d9-04a1-4d29-b605-72e2ffb1dc7f"
    value {
      scsi_controller {
        io_queue_depth: 128
        max_sub_channel_count: 4
        disks {
          key: 0
          value {
            disk { host_path: "/data/data.vhdx" format: DISK_FORMAT_VHDX }
          }
        }
      }
    }
  }

  # Serial console on COM1
  chipset_devices {
    isa_serial {
      com_port: COM_PORT_COM1
      backend { socket { path: "/tmp/console.sock" } }
    }
  }
}
```

---

## Hot-Plug / Hot-Remove Examples (ModifyDeviceRequest)

### 9. Hot-plug a VirtIO block device into an empty PCI port

```protobuf
modify_device_request {
  type: MODIFY_TYPE_ADD
  pci {
    bus_name: "pci0"
    port_name: "hotplug0"
    virtio {
      blk {
        disk {
          host_path: "/data/new-disk.vhdx"
          format: DISK_FORMAT_VHDX
        }
        read_only: false
      }
    }
  }
}
```

### 10. Hot-remove a PCI device

```protobuf
modify_device_request {
  type: MODIFY_TYPE_REMOVE
  pci {
    bus_name: "pci0"
    port_name: "hotplug0"
    # No device payload needed for REMOVE — port name is enough.
  }
}
```

### 11. Hot-add a SCSI disk to an existing controller

```protobuf
modify_device_request {
  type: MODIFY_TYPE_ADD
  scsi_disk {
    controller_instance_id: "ba6163d9-04a1-4d29-b605-72e2ffb1dc7f"
    lun: 1
    device {
      read_only: false
      disk {
        host_path: "/data/extra-data.vhdx"
        format: DISK_FORMAT_VHDX
      }
    }
  }
}
```

### 12. Hot-remove a SCSI disk

```protobuf
modify_device_request {
  type: MODIFY_TYPE_REMOVE
  scsi_disk {
    controller_instance_id: "ba6163d9-04a1-4d29-b605-72e2ffb1dc7f"
    lun: 1
    # No device payload needed — LUN identifies the disk.
  }
}
```

### 13. Hot-add an NVMe namespace to an existing controller

```protobuf
modify_device_request {
  type: MODIFY_TYPE_ADD
  nvme_namespace {
    bus_name: "pci0"
    port_name: "nvme-port"
    nsid: 3
    device {
      read_only: false
      disk {
        host_path: "/data/new-ns.raw"
        format: DISK_FORMAT_RAW
      }
    }
  }
}
```

### 14. Hot-add a VirtIO net device on MMIO

```protobuf
modify_device_request {
  type: MODIFY_TYPE_ADD
  mmio {
    bus_name: "mmio0"
    base_address: 285212672  # 0x11000000
    irq: 43
    virtio {
      net {
        mac_address: "de:ad:be:ef:00:01"
        max_queues: 2
        backend { tap { name: "tap0" } }
      }
    }
  }
}
```

---

## Reusable Leaf Types

The schema defines three shared leaf types that appear across all bus types:

| Leaf type | Used by |
|-----------|---------|
| `DiskConfig` | VirtioBlkDevice, NvmeNamespace, ScsiDisk |
| `NetworkBackend` | VirtioNetDevice, SynthNetDevice |
| `SerialBackend` | VirtioConsoleDevice, IsaSerialDevice, UartDevice |

This means adding a new bus type or storage controller never requires
redefining disk formats, network backends, or serial backends.

## Device Hierarchy Summary

```
DevicesConfig
├── pci_buses (map<string, PciBus>)
│   └── root_ports (map<string, PciRootPort>)
│       └── PciDevice
│           ├── VirtioDevice → (blk, net, fs, pmem, console, vsock)
│           └── NvmeDevice → namespaces (map<uint32, NvmeNamespace>)
├── mmio_buses (map<string, MmioBus>)
│   └── devices (map<uint64, MmioDevice>)
│       ├── VirtioDevice → (same as PCI)
│       └── UartDevice
├── vmbus_devices (map<string, VmbusDevice>)
│   ├── ScsiControllerDevice → disks[] (ScsiDisk)
│   └── SynthNetDevice
└── chipset_devices[]
    └── IsaSerialDevice
```
