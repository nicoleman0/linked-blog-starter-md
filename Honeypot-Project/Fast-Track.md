Here's your fast-track plan:

1. **This week**: Spin up a cheap cloud VM, install Conpot, deploy basic Modbus template, start logging
2. **Verify it's working**: Use Shodan to confirm your honeypot appears in search results (they scan the entire IPv4 space regularly)
3. **Iterate while collecting**: You can refine your honeypot implementation over the next few weeks while data is already accumulating

The beauty of honeypots is you can improve them while they're running - initial data collection doesn't need perfection, just something credible enough to attract attention.

**Quick start stack:**

- Ubuntu 22.04 LTS VM
- Conpot (gives you baseline Modbus honeypot in minutes)
- ELK stack or even just structured logging to files
- Daily backup of logs