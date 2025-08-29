# Ruby LLM Client - Product Requirements Document

## Executive Summary

Ruby LLM Client is a unified, DSL-driven Ruby library that provides a consistent interface for interacting with multiple Large Language Model providers (OpenAI, Anthropic, Cohere, etc.). By leveraging Ruby's metaprogramming capabilities, it offers an elegant, expressive API that abstracts away provider-specific implementation details while maintaining full access to advanced features.

**Vision**: To become the standard Ruby library for LLM integration, enabling developers to switch between providers seamlessly and focus on building applications rather than managing API differences.

## User Stories

### Core Developer Experience

**As a Ruby developer, I want to:**
- Configure multiple LLM providers with a clean, declarative syntax so I can switch between them easily
- Send chat messages using a natural, block-based DSL so my code is readable and maintainable
- Switch between different providers without changing my application logic so I can optimize for cost/performance
- Handle errors consistently across all providers so I don't need provider-specific error handling
- Stream responses in real-time with a unified interface so I can build responsive applications

**As a DevOps engineer, I want to:**
- Configure providers using environment variables and configuration files so I can manage secrets securely
- Monitor usage and costs across multiple providers so I can optimize spending
- Set rate limits and retry policies consistently so I can prevent API abuse
- Log requests and responses uniformly so I can debug issues effectively

**As a data scientist, I want to:**
- Generate embeddings from multiple providers with the same interface so I can compare quality
- Batch process requests efficiently so I can handle large datasets
- Access raw provider responses when needed so I can analyze provider-specific metadata
- Cache responses automatically so I can avoid redundant API calls during experimentation

### Advanced Use Cases

**As an application developer, I want to:**
- Use function calling/tools with a consistent interface across providers
- Process images and documents through vision-enabled models uniformly
- Create conversation templates that work across all providers
- A/B test different models transparently in production

**As a library maintainer, I want to:**
- Add new providers with minimal code changes so the library stays current
- Extend functionality through plugins so the core remains lightweight
- Provide comprehensive documentation and examples so adoption is smooth

## Functional Requirements

### FR1: Provider Management
- **FR1.1**: Register multiple LLM providers (OpenAI, Anthropic, Cohere, Google, etc.)
- **FR1.2**: Configure provider-specific settings (API keys, base URLs, default models)
- **FR1.3**: Set default provider for simplified usage
- **FR1.4**: Override provider on a per-request basis

```ruby
# Configuration DSL
LLM.configure do
  provider :openai do
    api_key ENV['OPENAI_API_KEY']
    default_model 'gpt-4'
    timeout 30
  end

  provider :anthropic do
    api_key ENV['ANTHROPIC_API_KEY']  
    default_model 'claude-3-sonnet-20240229'
  end

  default_provider :openai
end
```

### FR2: Chat Interface
- **FR2.1**: Support structured conversation building with system, user, and assistant messages
- **FR2.2**: Handle single-turn and multi-turn conversations
- **FR2.3**: Provide method chaining for fluent API usage
- **FR2.4**: Support conversation templates and presets

```ruby
# Basic chat
response = LLM.chat do
  system "You are a helpful coding assistant"
  user "Explain Ruby metaprogramming"
  temperature 0.7
  max_tokens 500
end

# Provider-specific call
response = LLM.anthropic do
  user "What's the weather like?"
  model 'claude-3-haiku-20240307'
end

# Conversation building
conversation = LLM.conversation do
  system "You are a code reviewer"
  user "Here's my Ruby code..."
  assistant "I see a few issues..."
  user "Can you fix them?"
end

response = conversation.send_to(:openai)
```

### FR3: Streaming Support
- **FR3.1**: Enable real-time streaming for all supported providers
- **FR3.2**: Provide callbacks for handling streamed chunks
- **FR3.3**: Support both block-based and enumerator-based streaming

```ruby
# Block-based streaming
LLM.chat stream: true do
  user "Write a story about Ruby"
  on_chunk { |chunk| print chunk }
  on_complete { |full_response| puts "\n--- Complete ---" }
end

# Enumerator streaming
stream = LLM.openai(stream: true) { user "Count to 10" }
stream.each { |chunk| puts chunk }
```

### FR4: Response Processing
- **FR4.1**: Normalize response format across providers
- **FR4.2**: Provide access to raw provider responses
- **FR4.3**: Extract common metadata (tokens used, model, finish reason)
- **FR4.4**: Handle partial responses and errors gracefully

```ruby
response = LLM.chat { user "Hello" }

puts response.content        # Normalized content
puts response.tokens_used    # Usage information
puts response.model         # Model that generated response
puts response.provider      # Provider used
puts response.raw           # Raw provider response
```

### FR5: Advanced Features
- **FR5.1**: Function calling with unified interface
- **FR5.2**: Multi-modal support (text, images, documents)
- **FR5.3**: Embeddings generation
- **FR5.4**: Batch processing capabilities

```ruby
# Function calling
response = LLM.chat do
  user "What's the weather in Tokyo?"
  functions [
    {
      name: 'get_weather',
      description: 'Get weather for a location',
      parameters: {
        type: 'object',
        properties: {
          location: { type: 'string' }
        }
      }
    }
  ]
  on_function_call { |call| handle_weather_request(call) }
end

# Multi-modal
response = LLM.chat do
  user do
    text "What's in this image?"
    image "/path/to/image.jpg"
  end
end

# Embeddings
embeddings = LLM.embed("Ruby is a great language")
# or
embeddings = LLM.openai.embed(["text1", "text2", "text3"])
```

## Non-Functional Requirements

### NFR1: Performance
- **NFR1.1**: Response time overhead < 10ms for request preparation
- **NFR1.2**: Memory usage scales linearly with conversation length
- **NFR1.3**: Support concurrent requests without blocking
- **NFR1.4**: Implement connection pooling for HTTP requests

### NFR2: Reliability
- **NFR2.1**: Automatic retry with exponential backoff for transient failures
- **NFR2.2**: Circuit breaker pattern for provider outages
- **NFR2.3**: Request timeout configuration per provider
- **NFR2.4**: Graceful degradation when providers are unavailable

### NFR3: Extensibility
- **NFR3.1**: Plugin architecture for adding new providers
- **NFR3.2**: Middleware system for request/response transformation
- **NFR3.3**: Custom serializers and deserializers
- **NFR3.4**: Hook system for monitoring and logging

### NFR4: Security
- **NFR4.1**: Secure credential management
- **NFR4.2**: Request/response logging with PII redaction
- **NFR4.3**: SSL/TLS verification for all HTTP requests
- **NFR4.4**: Rate limiting to prevent API abuse

## Technical Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Application                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    LLM Client DSL                          │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │
│  │ Config DSL  │ │  Chat DSL    │ │   Streaming DSL     │   │
│  └─────────────┘ └──────────────┘ └─────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                Request Router                               │
│             (method_missing magic)                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                Provider Adapters                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ OpenAI   │ │Anthropic │ │  Cohere  │ │    Google    │   │
│  │ Adapter  │ │ Adapter  │ │ Adapter  │ │   Adapter    │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                HTTP Client Layer                            │
│      (Net::HTTP / Faraday with connection pooling)         │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

1. **DSL Layer**: Provides the user-facing API with metaprogramming
2. **Request Router**: Dispatches requests to appropriate providers
3. **Provider Adapters**: Transform requests/responses for each provider
4. **HTTP Client**: Handles actual API communication
5. **Middleware System**: Extensible request/response processing

## Implementation Roadmap

### Phase 1: Core Foundation (2-3 weeks)
**Epic: Basic LLM Client Infrastructure**

**Tasks:**
1. **Setup project structure and dependencies**
   - Initialize gem structure with proper gemspec
   - Add development dependencies (rspec, rubocop, yard)
   - Setup CI/CD pipeline with GitHub Actions
   - Create basic documentation structure

2. **Implement core DSL framework**
   - Create base LLMClient class with metaprogramming hooks
   - Implement provider registration system
   - Build configuration DSL using class-level methods
   - Add method_missing delegation to providers

3. **Create provider adapter pattern**
   - Define abstract Provider class
   - Implement request/response transformation interface
   - Create configuration validation system
   - Build provider capability detection

4. **Implement OpenAI adapter**
   - Map DSL to OpenAI chat completions API
   - Handle authentication and headers
   - Transform request/response formats
   - Add error handling and status code mapping

5. **Add basic chat functionality**
   - Implement conversation building DSL
   - Support system, user, assistant message roles
   - Add parameter configuration (temperature, max_tokens)
   - Create response normalization

**Acceptance Criteria:**
- Can configure OpenAI provider
- Can send basic chat requests
- Receives normalized responses
- Error handling works for common failures

```ruby
# Should work by end of Phase 1
LLM.configure do
  provider :openai do
    api_key ENV['OPENAI_API_KEY']
    default_model 'gpt-4'
  end
  default_provider :openai
end

response = LLM.chat do
  system "You are helpful"
  user "Hello world"
  temperature 0.7
end

puts response.content
puts response.tokens_used
```

### Phase 2: Multi-Provider Support (1-2 weeks)
**Epic: Provider Ecosystem**

**Tasks:**
6. **Add Anthropic adapter**
   - Implement Anthropic Messages API mapping
   - Handle authentication differences
   - Map parameter names and constraints
   - Test compatibility with existing DSL

7. **Implement provider switching**
   - Allow per-request provider override
   - Add provider-specific method routing
   - Validate provider capabilities before requests
   - Handle provider-specific limitations gracefully

8. **Create response normalization**
   - Standardize response object structure
   - Map provider-specific metadata
   - Handle different error response formats
   - Preserve access to raw responses

9. **Add comprehensive error handling**
   - Create unified exception hierarchy
   - Map provider-specific errors to common types
   - Implement retry logic with exponential backoff
   - Add circuit breaker for failed providers

**Acceptance Criteria:**
- OpenAI and Anthropic work with same DSL
- Can switch providers mid-application
- Errors are handled consistently
- Raw responses accessible when needed

### Phase 3: Advanced Features (2-3 weeks)
**Epic: Production-Ready Capabilities**

**Tasks:**
10. **Implement streaming support**
    - Add streaming DSL with callbacks
    - Handle server-sent events parsing
    - Implement both block and enumerator interfaces
    - Test streaming across providers

11. **Add function calling support**
    - Create function definition DSL
    - Handle function call responses
    - Implement callback system for function execution
    - Map provider-specific function formats

12. **Implement batch processing**
    - Add batch request DSL
    - Handle concurrent requests with limits
    - Implement result aggregation
    - Add progress tracking for large batches

13. **Create embeddings interface**
    - Add embedding generation methods
    - Support batch embedding requests
    - Normalize embedding response formats
    - Add similarity comparison utilities

**Acceptance Criteria:**
- Streaming works across providers
- Function calling integrates seamlessly
- Batch processing handles errors gracefully
- Embeddings provide consistent interface

### Phase 4: Developer Experience (1-2 weeks)
**Epic: Polish and Documentation**

**Tasks:**
14. **Add comprehensive testing**
    - Unit tests for all DSL components
    - Integration tests with real APIs
    - Mock provider for testing applications
    - Performance benchmarks

15. **Create extensive documentation**
    - API reference with YARD
    - Getting started guide
    - Provider-specific configuration guides
    - Example applications and use cases

16. **Implement monitoring and debugging**
    - Add request/response logging
    - Implement performance metrics
    - Create debugging helpers
    - Add health check endpoints

17. **Package and release preparation**
    - Optimize gem dependencies
    - Create release scripts
    - Prepare RubyGems.org publication
    - Setup semantic versioning

## Success Metrics

### Adoption Metrics
- **Downloads**: 1,000+ gem downloads within 3 months
- **GitHub Stars**: 100+ stars within 6 months  
- **Community Usage**: 5+ community-contributed providers within 1 year

### Technical Metrics
- **Performance**: <10ms overhead for request preparation
- **Reliability**: 99.9% uptime for supported providers
- **Coverage**: 90%+ test coverage across all components

### Developer Experience Metrics
- **Documentation**: Complete API documentation with examples
- **Time to First Success**: <5 minutes from gem install to first successful request
- **Provider Addition**: New provider can be added in <100 lines of code

## Risk Assessment

### High Risk
- **Provider API Changes**: Major changes to provider APIs could break compatibility
  - *Mitigation*: Version provider adapters, maintain backward compatibility
- **Rate Limiting**: Aggressive rate limiting could impact usability
  - *Mitigation*: Implement intelligent retry and queue management

### Medium Risk  
- **Performance Overhead**: Ruby metaprogramming could add latency
  - *Mitigation*: Benchmark and optimize critical paths
- **Memory Usage**: Large conversations could consume significant memory
  - *Mitigation*: Implement conversation size limits and streaming

### Low Risk
- **Gem Dependencies**: Dependency conflicts in larger applications
  - *Mitigation*: Minimize dependencies, use conservative version constraints

## Open Questions

1. **Pricing Integration**: Should the library track and report costs across providers?
2. **Caching Strategy**: What level of response caching should be built-in?
3. **Async Support**: Should we provide async/await style interfaces using Fiber?
4. **Configuration Management**: How should production deployments manage multiple provider credentials?
5. **Plugin Ecosystem**: What's the best way to enable community-contributed providers?

---

*This PRD is a living document that will be updated based on user feedback, technical discoveries, and market changes during implementation.*